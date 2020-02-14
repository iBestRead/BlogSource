---
title: 使用Serilog记录路由的端点
tags: 
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 日志
  - Serilog

date: 2020-02-14 07:02:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Logging the selected Endpoint Name with Serilog](https://andrewlock.net/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是[在ASP.NET Core 3.0中使用Serilog](/using-serilog-aspnetcore-in-asp-net-core-3/)第2篇:

1. [使用Serilog减少日志的详细程度](/using-serilog-aspnetcore-in-asp-net-core-3-reducing-log-verbosity/)
4. [使用Serilog记录路由的端点](/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/)
5. [使用Serilog记录MVC属性](/using-serilog-aspnetcore-in-asp-net-core-3-logging-mvc-propertis-with-serilog/)
6. [在Serilog日志中排除健康检查日志](/using-serilog-aspnetcore-in-asp-net-core-3-excluding-health-check-endpoints-from-serilog-request-logging/)

在本系列的[上一篇](/using-serilog-aspnetcore-in-asp-net-core-3-reducing-log-verbosity/)中，我介绍了如何使用Serilog的`RequestLogging`中间件，对每次请求只生成“摘要”日志，替换掉由ASP.NET Core日志组件输出的10多行的日志。

在本文中，我将介绍如何向Serilog的摘要日志中添加其它额外的元数据，例如，请求方的主机名、响应的内容类型、ASP.NET Core 3.0的[端点路由中间件]( https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.0#endpoint-routing-differences-from-earlier-versions-of-routing )中调用的端点。

<!-- more -->

## ASP.NET Core默认日志很详细但也很啰嗦

[上一篇文章](/using-serilog-aspnetcore-in-asp-net-core-3-reducing-log-verbosity/)中有谈到，在开发环境中，一次请求，ASP.NET Core生成10条日志消息：

![](https://cdn.ibestread.com/img/logs_before_serilog.png)

使用*Serilog.AspNetCore*的`RequestLoggingMiddleware`中间件，可以缩减为一条日志消息：

![](https://cdn.ibestread.com/img/logs_after_serilog.png)

> 上面的截图均来至[Seq](https://datalust.co/seq)，它可以方便的展现结构化日志

显然，原始日志集更加冗长，其中大部分都是无用信息。 但是把原始的10条日志看作1条的话，它比Serilog摘要日志多了些字段。

我们来看看额外添加了哪些字段：

- Host (`localhost:5001`) 
- Scheme (`https`) 
- Protocol (`HTTP/2`) 
- QueryString (`test=true`) 
- EndpointName (`/Index`) 
- HandlerName (`OnGet`/`SerilogRequestLogging.Pages.IndexModel.OnGet`) 
- ActionId (`1fbc88fa-42db-424f-b32b-c2d0994463f1`) 
- ActionName (`/Index`) 
- RouteData (`{page = "/Index"}`) 
- ValidationState (`True`/`False`) 
- ActionResult (`PageResult`) 
- ContentType (`text/html; charset=utf-8`) 

个人认为，这些额外添加的字段还是很有用的。例如，如果您的应用程序绑定了多个主机，那么“`Host`”对于日志记录绝对重要。 `QueryString`平常也会用到。 `EndpointName`、`HandlerName`、`ActionId`、`ActionName`则不太重要，因为可以通过`RequestPath`推断出来，但是当程序产生异常时，单独记录这些额外字段有助于我们定位问题。

我们可以将这些信息分为两大类：

- 请求/响应类。 `Host`, `Scheme`, `ContentType`, `QueryString`, `EndpointName` 
- MVC/Razor类。 `HandlerName`, `ActionId`, `ActionResult` 

在本文章中，我将介绍如何将第一类的信息添加到日志中。在下一篇中再介绍如何将第二类的信息添加到日志中。

## 添加额外信息到Serilog日志

在上一篇文章中，已经介绍了应用程序添加Serilog及开启`RequestLogging`中间件。 假设您已经按照上一篇的步骤，完成了`Startup.Configure`代码：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... Error handling/HTTPS middleware
    app.UseStaticFiles();

    app.UseSerilogRequestLogging(); // <-- Add this line

    app.UseRouting();
    app.UseAuthorization();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapRazorPages();
    });
}
```

将Serilog的`RequestLoggingMiddleware`添加到管道中，使用`UseSerilogRequestLogging()`扩展方法，这个扩展方法有几个重载，用来[配置参数RequestLoggingOptions](https://github.com/serilog/serilog-aspnetcore/blob/ffed9d231aefc3de7c13a03a570fb45c326632b0/src/Serilog.AspNetCore/AspNetCore/RequestLoggingOptions.cs)。 `RequestLoggingOptions`定义如下：

```csharp
public class RequestLoggingOptions
{
    public string MessageTemplate { get; set; }
    public Func<HttpContext, double, Exception, LogEventLevel> GetLevel { get; set; }
    public Action<IDiagnosticContext, HttpContext> EnrichDiagnosticContext { get; set; }
}
```

其中的属性`MessageTemplate`输出日志的字符串模板，而`GetLevel`属性是设置日志的输入级别 `Debug` / `Info` / `Warning`等。属性`EnrichDiagnosticContext`是本文关注的。

在生成日志消息时，此`Action<>`被Serilog的中间件所调用。 它在中间件管道**执行之后**，日志**写入之前**运行。 例如，在下图中（取自我的书《[ASP.NET Core in Action](https://www.manning.com/books/asp-dot-net-core-in-action?a_aid=aspnetcore-in-action&a_bid=5b1b11eb)》），当响应“回传”中间件管道时，在第5步写入日志：

![](https://cdn.ibestread.com/img/03_01.png)

在管道处理之后再写入日志，说明了两件事：

- 可以访问到*Response*的相关属性，例如状态代码、经过的时间、内容类型
- 可以访问到后续管道中间件配置的*Features*，例如由`EndpointRoutingMiddleware`设置的`IEndpointFeature`（由`UseRouting()`添加）。

下一小节，我将编写一个帮助类，实现将所有“缺少”属性添加到Serilog日志消息中。

## 在IDiagnosticContext添加额外属性

*Serilog.AspNetCore*将接口`IDiagnosticContext`作为单例添加到DI容器，您可以从其它类注入使用。 通过调用`Set()`将其它额外属性添加到日志消息中。

例如，[如文档中所示](https://github.com/serilog/serilog-aspnetcore#request-logging)，您可以添加任意属性：

```csharp
public class HomeController : Controller
{
    readonly IDiagnosticContext _diagnosticContext;
    public HomeController(IDiagnosticContext diagnosticContext)
    {
        _diagnosticContext = diagnosticContext;
    }

    public IActionResult Index()
    {
        // The request completion event will carry this property
        _diagnosticContext.Set("CatalogLoadTime", 1423);
        return View();
    }
}
```

那么输出的摘要日志将包含`CatalogLoadTime`属性。

我们可以借助上面的思想，通过对`RequestLoggingOptions.EnrichDiagnosticContext`，对`IDiagnosticContext`进行额外属性添加。

```csharp
public static class LogHelper 
{
    public static void EnrichFromRequest(IDiagnosticContext diagnosticContext, HttpContext httpContext)
    {
        var request = httpContext.Request;

        // Set all the common properties available for every request
        diagnosticContext.Set("Host", request.Host);
        diagnosticContext.Set("Protocol", request.Protocol);
        diagnosticContext.Set("Scheme", request.Scheme);

        // Only set it if available. You're not sending sensitive data in a querystring right?!
        if(request.QueryString.HasValue)
        {
            diagnosticContext.Set("QueryString", request.QueryString.Value);
        }

        // Set the content-type of the Response at this point
        diagnosticContext.Set("ContentType", httpContext.Response.ContentType);

        // Retrieve the IEndpointFeature selected for the request
        var endpoint = httpContext.GetEndpoint();
        if (endpoint is object) // endpoint != null
        {
            diagnosticContext.Set("EndpointName", endpoint.DisplayName);
        }
    }
}
```

上面的帮助类从`HttpContext`的属性`Request`、`Response`、`GetEndpoint()`中获取相关属性。 

修改`Startup.Configure()`方法，在调用`UseSerilogRequestLogging`时设置`EnrichDiagnosticContext`属性：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ... Other middleware

    app.UseSerilogRequestLogging(opts
        => opts.EnrichDiagnosticContext = LogHelper.EnrichFromRequest);

    // ... Other middleware
}
```

此时，我们再来看看日志消息，所有额外添加的属性都被Serilog正常记录了：

![](https://cdn.ibestread.com/img/logs_after_serilog_endriched.png)

通常使用了`HttpContext`的中间件管道，就可以采用上面方法来记录额外属性。 但是有个例外，就是MVC中的特定功能，这些功能是MVC中间件“内部”实现的，例如*Action*名称或*RazorPage*处理程序名称。 在下一篇文章中，我将展示如何将它们也添加到Serilog日志中来。

## 总结

默认情况下，用Serilog的Request日志记录中间件替换ASP.NET Core日志时，与开发环境的默认日志相比，会丢失一些信息。 在本文中，我介绍了如何使用自定义Serilog的`RequestLoggingOptions`来添加这些其他属性。

`HttpContext`可以提供请求过程中的相关属性，通过`HttpContext`将需添加的字段，赋值给`IDiagnosticContext`中的属性。 那么这些附加属性会输出到Serilog生成的结构化日志中。 在下一篇文章中，我将介绍如何将MVC特定功能的属性添加到Request日志中。

