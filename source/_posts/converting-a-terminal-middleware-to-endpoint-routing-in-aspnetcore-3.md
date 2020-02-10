---
title: 将末端中间件转换为端点路由
tags: 
  - .NET CORE
  - .NET CORE 3.0
  - 端点路由
  - Endpoint Routing

date: 2020-02-10 07:40:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Converting a terminal middleware to endpoint routing in ASP.NET Core 3.0](https://andrewlock.net/converting-a-terminal-middleware-to-endpoint-routing-in-aspnetcore-3/)
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [升级至 ASP.NET Core 3.0](/upgrading-to-asp-net-core-3) 第3篇:

1. [转换.NET Standard 2.0类库到.NET Core 3.0](/converting-a-netstandard-2-library-to-netcore-3/)
2. [对比IHostingEnvironment与IHostEnvironment .NET及Core 3.0中的过时类型](/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/)
3. [不要在Startup类的构造函数中使用依赖注入](/avoiding-startup-service-injection-in-asp-net-core-3/)
4. [将末端中间件转换为端点路由](/converting-a-terminal-middleware-to-endpoint-routing-in-aspnetcore-3/)
5. [将集成测试升级至.NET Core 3.0](/converting-integration-tests-to-net-core-3/)

在这篇文章中，主要介绍端点路由，并演示如何创建一个响应URL请求的端点。 并展示如何将ASP.NET Core 2.x中的末端中间件，升级为ASP.NET Core 3.0中的端点路由。

<!-- more -->

##  路由的演变 

[ASP.NET Core中的路由](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.0)是将请求URL路径（例如/Orders/1）映射到响应的处理程序的过程。 它主要与MVC中间件一起使用，以将请求映射到*Controllers*和*Actions*。 还可以反向映射，根据指定参数生成URL。

在ASP.NET Core 2.1和更低版本中，通过实现[IRouter接口](https://docs.microsoft.com/dotnet/api/microsoft.aspnetcore.routing.irouter)将请求的URL映射到处理程序来处理路由。 通常，不会直接实现这个接口来处理路由，而是在管道中添加`MvcMiddleware`实现。 一旦请求到达`MvcMiddleware`，路由会确定传入请求URL应该由哪个*Controller*的*Action*来执行。

另外，在执行*Action*前会经过了各种[MVC过滤器](https://andrewlock.net/asp-net-core-in-action-filters/)。 这些过滤器形成了另一条管道。在一些情况下，我们会重复一些中间件的行为。 一个典型的例子就是[CORS政策](https://docs.microsoft.com/en-us/aspnet/core/security/cors?view=aspnetcore-2.2)。 为了对不同的*MVC Action*配置不同的CORS策略，肯定会有些重复的代码。

中间件管道的“分支”通常用于“伪路由”。在中间件管道中使用[Map()方法](https://github.com/aspnet/AspNetCore/blob/v2.1.12/src/Http/Http.Abstractions/src/Extensions/MapExtensions.cs)，当请求的Url前缀满足条件时，执行指定的中间件。

例如，在*Startup.cs*中的`Configure()`方法对管道进行分支，当传入路径为`/ping`时，末端（很多会翻译为终端，我觉得终端有歧义）中间件将执行：

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseStaticFiles();
    app.UseCors();
    app.Map("/ping", 
        app2 => app2.Run(async context =>
        {
            await context.Response.WriteAsync("Pong");
        });
    app.UseMvcWithDefaultRoute();
}
```

在这种情况下，`Run()`方法是末端中间件，因为它直接返回响应。这个Map分支对应于应用程序来说，`app2`没有做其它事，就是一个末端。

![](https://cdn.ibestread.com/img/branching_middleware.svg)

与`MvcMiddleware`中的端点（*Controller*的*Action*）相比，该末端有点像是二等公民。 从传入路由中解析对象比较麻烦，您必须自己手动实现授权。

另一个问题是，例如，当请求到达`UseCors(`)中间件时，我们需要知道在哪个分支或末端上运行，`/ping`端点允许跨域请求，而MVC中间件则不允许 。

在ASP.NET Core 2.2中，Microsoft引入了[端点路由](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.0)作为MVC控制器的新路由机制。 它的实现是在`MvcMiddleware`内部，还无法解决上述问题。 但是在ASP.NET Core 3.0中，它的范围有所扩展，成为了主要的路由机制。

新的端点路由将请求的路由与处理程序的实际执行分开。 这意味着您可以提前知道哪个处理程序将被执行。 

![](https://cdn.ibestread.com/img/endpoint_routing.svg)

如何将之前的ping-pong管道映射到新的端点路由方式？ 

## 在ASP.NET Core 2.x中使用Map()时

我们来看一个[的自定义中间件](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/write?view=aspnetcore-2.2)，它只[返回应用程序的版本信息](https://andrewlock.net/version-vs-versionsuffix-vs-packageversion-what-do-they-all-mean/#fileversion)，它是一个末端中间件，从代码中可以看到它没有再向后调用`_netx`。

```csharp
public class VersionMiddleware
{
    readonly RequestDelegate _next;
    static readonly Assembly _entryAssembly = System.Reflection.Assembly.GetEntryAssembly();
    static readonly string _version = FileVersionInfo.GetVersionInfo(_entryAssembly.Location).FileVersion;

    public VersionMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        context.Response.StatusCode = 200;
        await context.Response.WriteAsync(_version);

        //we're all done, so don't invoke next middleware
    }
}
```

在ASP.NET Core 2.x中，您可以在*Startup.cs*中通过使用`Map()`扩展方法来指定URL和对应的中间件：

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseStaticFiles();

    app.UseCors();

    app.Map("/version", versionApp => versionApp.UseMiddleware<VersionMiddleware>()); 

    app.UseMvcWithDefaultRoute();
}
```

当您使用`/version`开头的URL（`/version`或`/version/test`）访问时，都获得相同的响应：

```bash
1.0.0
```

当您发送带有任何（除静态文件以外）的URL的请求时，将调用MvcMiddleware，并处理该请求。 使用上面的代码，CORS中间件无法知道哪个端点将最终被执行。

## 改中间件为端点路由

在ASP.NET Core 3.0中，我们使用端点路由，路由与端点的调用是分开的。 实际上，是有两个中间件：

- `EndpointRoutingMiddleware` 称为路由中间件。根据请求的URL来计算由哪个端点来执行。
- `EndpointMiddleware`  成为端点中间件，调用中间件。

它们分别添加在管道的两个不同的位置，起着两个不同的作用。 通常，您希望路由中间件更早的添加到管道中，以便后续的中间件可以获得端点（即将被执行地）的信息。 端点中间件应该在管道的最后。 例如：

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseStaticFiles();

    // Add the EndpointRoutingMiddleware
    app.UseRouting();

    // All middleware from here onwards know which endpoint will be invoked
    app.UseCors();

    // Execute the endpoint selected by the routing middleware
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapDefaultControllerRoute();
    });
}
```

扩展方法`UseRouting()`将`EndpointRoutingMiddleware`添加到管道，而扩展方法`UseEndpoints()`将`EndpointMiddleware`添加到管道。 通过使用`UseEndpoints()`会注册所有端点（在上面的示例中，我们仅注册我们的MVC控制器）。

> 注意：通常将静态文件中间件放在路由中间件之前。 这样可以避免在请求静态文件时有额外的路由开销。 [如迁移文档](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30?view=aspnetcore-3.0&tabs=visual-studio#migrate-startupconfigure)中所述，将身份验证和授权*controllers* 放在两个中间件中间也很重要。

下面我们使用新的端点路由方式改写之前的`VersionMiddleware`：

我们使用`/version` URL作为匹配路径，将注册版本端点的代码移到`UseEndpoints()`调用中：

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseStaticFiles();

    app.UseRouting();

    app.UseCors();

    app.UseEndpoints(endpoints =>
    {
        // Add a new endpoint that uses the VersionMiddleware
        endpoints.Map("/version", endpoints.CreateApplicationBuilder()
            .UseMiddleware<VersionMiddleware>()
            .Build())
            .WithDisplayName("Version number");

        endpoints.MapDefaultControllerRoute();
    });
}
```

有几个重点我们需要注意：

- 我们使用` IApplicationBuilder ()`来创建了`RequestDelegate`。
- 对路径的完整匹配，不再是前缀匹配方式。
- 可以对端点设置一个显示名称（上面代码中的"Version number"）
- 可以附加额外元数据（上面代码中没有展示）

`Map`方法需要`RequestDelegate`而不是`Action <IApplicationBuilder>`。导致添加中间件为端点的代码要比之前2.x更加冗长。 可以写一个扩展方法来解决这个问题：

```csharp
public static class VersionEndpointRouteBuilderExtensions
{
    public static IEndpointConventionBuilder MapVersion(this IEndpointRouteBuilder endpoints, string pattern)
    {
        var pipeline = endpoints.CreateApplicationBuilder()
            .UseMiddleware<VersionMiddleware>()
            .Build();

        return endpoints.Map(pattern, pipeline).WithDisplayName("Version number");
    }
}
```

然后在` Configure() `中使用这个扩展方法让代码更加简洁：

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseStaticFiles();

    app.UseRouting();

    app.UseCors();

    // Execute the endpoint selected by the routing middleware
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapVersion("/version");
        endpoints.MapDefaultControllerRoute();
    });
}
```

另外一个重要差异是，在ASP.NET Core 2.x中，`VersionMiddleware`匹配`/version`为前缀的所有请求。 比如`/version`，`/version/123`，`/version/test/oops`等等。当改为使用端点路由时，并不是前缀匹配，而是完整匹配。 可以在端点路由中都使用路由参数。 例如：

```csharp
endpoints.MapVersion("/version/{id:int?}");
```

这种写法可以匹配`/version`和`/version/123` URL，但不匹配`/version/test/oops`。 

端点路由的另外一个特性，可以将元数据库附加到端点上。

端点的另一个功能是可以将元数据附加到端点。 在前面的示例中，我们提供了一个显示名称（主要用于调试目的），您还可以附加更多信息，例如授权策略或CORS策略，供其它中间件查询。 例如：

```csharp
public void Configure(IApplicationBuilder app)
{
    app.UseStaticFiles();

    app.UseRouting();

    app.UseCors();
    app.UseAuthentication();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapVersion("/version")
            .RequireCors("AllowAllHosts")
            .RequireAuthorization("AdminOnly");

        endpoints.MapDefaultControllerRoute();
    });
}
```

上面代码中，我们向*Version*端点中添加了CORS策略（`AllowAllHosts`）和授权策略（`AdminOnly`）。 当请求到达时，路由中间件选择*Version*端点，并附带了相关元数据， 当授权中间件和CORS中间件，发现*Version*端点存在这些策略时，会在*Version*端点执行前，先执行授权中间件和CORS中间件的逻辑。

## 必须把中间件改为端点路由吗

不是必须的。中间件方式的管道概念并没有变。您仍然可以像ASP.NET Core 1.0以后的版本一样，完全从中间件分支或短路返回。不需要使用端点路由来替换原理的方法。

但是，使用端点路由有3个优势：

- 可以将元数据附加到端点，以便其它中间件（例如Authorization，CORS）可以知道最终端点是谁，该执行什么操作
- 可以在非MVC的端点中使用路由模板，因此可以使用路由解析这个特性
- 可以更方便地生成指向非MVC端点的URL

如果上述特性对您有用，那么端点路由非常适合您。 例如，[ASP.NET Core HealthCheck](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/health-checks?view=aspnetcore-3.0)已改为为端点路由，那么在请求健康状态Url时，添加授权检查。

如果您觉得这些特性没用，则没有理由不必转换为端点路由。 例如，静态文件中间件通常是短路响应的，没有将其转换为端点路由。 静态文件通常不需要将授权或CORS。

最重要的是，静态文件中间应将件放在路由中间件之前。 

总体而言，端点路由在以前的路由方法中增加了许多特性，您需要在升级时注意差异。 如果尚未安装，请务必查看[迁移指南](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30)，其中详细介绍了许多更改。

## 总结

在这篇文章中，我概述了ASP.NET Core中的路由及其发展历程，及**路由的请求**与**处理程序的执行**分离后，带来的一些优势。

我还展示了，如何将ASP.NET Core 2.x应用程序中使用的简单末端中间件，转换为ASP.NET Core 3.0中的端点。 所需的更改相对较小，但需要注意的是**支持路由参数的完整匹配**替换了**前缀匹配**。