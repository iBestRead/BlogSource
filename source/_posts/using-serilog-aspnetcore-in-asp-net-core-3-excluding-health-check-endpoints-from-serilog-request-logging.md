---
title: 在Serilog日志中排除健康检查日志
tags: 
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 日志
  - Serilog
  - 健康检查

date: 2020-02-17 23:48:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Excluding health check endpoints from Serilog request logging](https://andrewlock.net/using-serilog-aspnetcore-in-asp-net-core-3-excluding-health-check-endpoints-from-serilog-request-logging/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是[在ASP.NET Core 3.0中使用Serilog](/using-serilog-aspnetcore-in-asp-net-core-3/)第4篇:

1. [使用Serilog减少日志的详细程度](/using-serilog-aspnetcore-in-asp-net-core-3-reducing-log-verbosity/)
4. [使用Serilog记录路由的端点](/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/)
5. [使用Serilog记录MVC属性](/using-serilog-aspnetcore-in-asp-net-core-3-logging-mvc-propertis-with-serilog/)
6. [在Serilog日志中排除健康检查日志](/using-serilog-aspnetcore-in-asp-net-core-3-excluding-health-check-endpoints-from-serilog-request-logging/)

在本系列的前几篇文章中，我介绍了如何配置Serilog的RequestLogging中间件，向Serilog的请求日志中添加其它额外信息，例如请求主机名或所选的端点名称。 我还介绍了如何使用过滤器将MVC或RazorPage特定的属性添加到日志中。

在这篇文章中，我将介绍如何不记录指定的请求的日志。 

<!-- more -->

# 健康检查被一直调用

当我们的程序在Kubernetes中运行，Kubernetes使用了两种“健康检查”来检查我们的应用程序是否正常运行，包括活动性探针和就绪性探针。我们会配置探针向应用程序发出HTTP请求，以指示应用程序运行正常。

>  Kubernetes版本1.16有第三类型的探针，[启动探针](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-startup-probes)。 

ASP.NET Core 2.2+中提供的[检查检查]( https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-3.1 )端点非常适合这些探针。 每次请求时返回`200 OK`响应，以让Kubernetes知道您的应用程序没有崩溃。

在ASP.NET Core 3.x中，可以使用端点路由来配置运行健康检查。 在*Startup.cs*中调用`AddHealthChecks()`注册服务，并使用`MapHealthChecks()`添加运行健康检查终端点：

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // ..other service configuration

        services.AddHealthChecks(); // Add health check services
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
         // .. other middleware
         app.UseRouting();
         app.UseAuthorization();
         app.UseEndpoints(endpoints =>
         {
             endpoints.MapHealthChecks("/healthz"); //Add health check endpoint
             endpoints.MapControllers();
         });
     }
}
```

在上面的示例中，向`/healthz`发送请求将调用运行健康检查终端点。 由于没有配置其它的检查项，因此只要应用程序正在运行，将始终返回200响应：

![](https://cdn.ibestread.com/img/serilog_health_check.png)

这么做会产生一个的问题，Kubernetes会经常请求该端点。 请求频率可以配置，通常每10秒请求一次。 也可以设置一个较高的频率，以便Kubernetes可以快速重启有故障的Pod。

Kestrel每秒可以处理数百万个请求，因此不会有性能问题。 但是每个请求至少生成1个日志可能会很烦人。

主要问题是，记录健康检查请求返回`ok`的日志并没有什么用处，与我们的业务逻辑没有关系。在下一部分中，我将介绍如何在Serilog请求日志排除这些健康检查请求。

## 使用自定义日志级别

在[上一篇文章中](/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/)，介绍了如何在Serilog请求日志中添加[所选端点](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.0#endpoint-routing-differences-from-earlier-versions-of-routing)的信息。 在注册Serilog中间件时为`RequestLoggingOptions.EnrichDiagnosticContext`属性提供一个[自定义函数](using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/#在IDiagnosticContext添加额外属性)：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... Other middleware

    app.UseSerilogRequestLogging(opts
        // EnrichFromRequest helper function is shown in the previous post
        => opts.EnrichDiagnosticContext = LogHelper.EnrichFromRequest); 

    app.UseRouting();
    app.UseAuthorization();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapHealthChecks("/healthz"); //Add health check endpoint
        endpoints.MapControllers();
    });
}
```

`RequestLoggingOptions`有另一个属性`GetLevel`，该属性是`Func<>`，用于确定日志记录的级别。 默认情况下，[按下面方式设置](https://github.com/serilog/serilog-aspnetcore/blob/dev/src/Serilog.AspNetCore/SerilogApplicationBuilderExtensions.cs#L31-L36)：

```csharp
public static class SerilogApplicationBuilderExtensions
{
    static LogEventLevel DefaultGetLevel(HttpContext ctx, double _, Exception ex) =>
        ex != null
            ? LogEventLevel.Error 
            : ctx.Response.StatusCode > 499 
                ? LogEventLevel.Error 
                : LogEventLevel.Information;
}
```

判断请求是否有异常，无异常时设置日志级别为`Information`；有异常或者*Response*代码为`5xx`。 设置日志级别为`Error`。

可以修改请求日志的默认级别由`Information`改为`Debug`，创建与上面示例类似的方法：

```csharp
public static class LogHelper
{
    public static LogEventLevel CustomGetLevel(HttpContext ctx, double _, Exception ex) =>
        ex != null
            ? LogEventLevel.Error 
            : ctx.Response.StatusCode > 499 
                ? LogEventLevel.Error 
                : LogEventLevel.Debug; //Debug instead of Information
}
```

在调用`UseSerilogRequestLogging()`时设置日志级别：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... Other middleware

    app.UseSerilogRequestLogging(opts => {
        opts.EnrichDiagnosticContext = LogHelper.EnrichFromRequest;
        opts.GetLevel = LogHelper.CustomGetLevel; // Use custom level function
    });

    //... other middleware
}
```

请求日志将全部记录为`Debug`，除非发生错误（Seq的屏幕截图）：

![](https://cdn.ibestread.com/img/seq_debug.png)

配置Serilog时，通常会设置最低的日志级别。 例如，下面代码将默认最低级别设置为`Debug()`，并写入控制台接收器：

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .WriteTo.Console()
    .CreateLogger();
```

如果想过滤日志，那么简单方法就是，将日志级别低于`LoggerConfiguration`中设定的`MinimumLevel`。 如果使用最低的级别`Verbose`，那么会全部都记录。

通常我们不会使用`Verbose`级别来记录上游日志，这样就没针对性，大量的日志对于使用Serilog中间件会变得毫无意义！

我们只想修改针对健康检查请求的日志级别为`Verbose`，在下一节中，我将介绍如何只针对特定的请求进行修改，其它请求还是保持原来的日志级别。 

## 针对健康检查的请求设置自定义日志级别

这里的关键是在写入日志时，能判断出健康检查请求。 如上所示，`GetLevel()`方法将当前的`HttpContext`作为参数，那么我们可以使用下面2种方法：

- 对比`HttpContext.Request`的`Path`与健康检查的`Path`
- 用所选的端点元数据来判断是否为健康检查端点触发的

第一种方法是最明显的，但是后面的处理会越来越麻烦，因此在这里我将跳过该方法。

第二个方法与[上一篇文章](using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/#在IDiagnosticContext添加额外属性)类似，在该文章中我们获得了`EndpointRoutingMiddleware`为给定请求选择的`IEndpointFeature`。 此功能提供了详细信息，例如所选端点的显示名称和路由数据。

假设[健康检查使用默认的“Health checks”显示名称注册](https://github.com/aspnet/AspNetCore/blob/master/src/Middleware/HealthChecks/src/Builder/HealthCheckEndpointRouteBuilderExtensions.cs#L18)，那么可以使用`HttpContext`判断“Health checks”请求，如下所示：

```csharp
public static class LogHelper
{
    private static bool IsHealthCheckEndpoint(HttpContext ctx)
    {
        var endpoint = ctx.GetEndpoint();
        if (endpoint is object) // same as !(endpoint is null)
        {
            return string.Equals(
                endpoint.DisplayName, 
                "Health checks",
                StringComparison.Ordinal);
        }
        // No endpoint, so not a health check endpoint
        return false;
    }
}
```

可以将此功能与[默认的GetLevel方法](https://github.com/serilog/serilog-aspnetcore/blob/dev/src/Serilog.AspNetCore/SerilogApplicationBuilderExtensions.cs#L31-L36)的自定义版本结合，以确保健康检查请求的日志级别为`Verbose`，而发生错误时使用`Error`，其它情况下使用`Information`：

```csharp
public static class LogHelper
{
    public static LogEventLevel ExcludeHealthChecks(HttpContext ctx, double _, Exception ex) => 
        ex != null
            ? LogEventLevel.Error 
            : ctx.Response.StatusCode > 499 
                ? LogEventLevel.Error 
                : IsHealthCheckEndpoint(ctx) // Not an error, check if it was a health check
                    ? LogEventLevel.Verbose // Was a health check, use Verbose
                    : LogEventLevel.Information;
        }
}
```

 剩下的就是更新Serilog中间件`RequestLoggingOptions`：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... Other middleware

    app.UseSerilogRequestLogging(opts => {
        opts.EnrichDiagnosticContext = LogHelper.EnrichFromRequest;
        opts.GetLevel = LogHelper.ExcludeHealthChecks; // Use the custom level
    });

    //... other middleware
}
```

运行应用程序后查看日志，只会看到标准请求的普通请求日志，没有健康检查的日志（除非发生错误！）。 下面截图中，我将Serilog改为记录`Verbose`级别，以便您可以查看健康检查的请求（通常应将它们滤掉过）

![](https://cdn.ibestread.com/img/seq_verbose.png)

## 总结

在这篇文章中，我介绍了如何为Serilog中间件的`RequestLoggingOptions`指定一个自定义函数，该函数可以设置指定请求日志的`LogEventLevel`。 例如，如何将其更改为`Debug`的默认级别。 如果您设定的级别低于`MinimumLevel`，那么指定请求的日志会被过滤掉不会记录。

另外还介绍了可以使用这种方法来过滤健康检查的请求日志。 通常，只有在应用程序有问题的情况下，这些请求才是有用的，但是请求成功也会生成日志。健康检查非常频繁，日志写入量也会显著增加。

这篇文章中的方法是，判断所选的`IEndpointFeature`的显示名称是否为“Health checks”。 如果是，则将以`Verbose`级别写入请求日志，该级别通常会被过滤掉。 