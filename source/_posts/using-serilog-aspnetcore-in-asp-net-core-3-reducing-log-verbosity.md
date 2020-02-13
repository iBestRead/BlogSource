---
title: 使用Serilog减少日志的详细程度
tags: 
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 日志
  - Serilog

date: 2020-02-13 08:55:00
---

> 译者:  [Akini Xu](/)
>
> 原文:  [Reducing log verbosity with Serilog RequestLogging](https://andrewlock.net/using-serilog-aspnetcore-in-asp-net-core-3-reducing-log-verbosity/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是[在ASP.NET Core 3.0中使用Serilog](/using-serilog-aspnetcore-in-asp-net-core-3/)第1篇:

1. [使用Serilog减少日志的详细程度](/using-serilog-aspnetcore-in-asp-net-core-3-reducing-log-verbosity/)
4. [使用Serilog记录所选的端点](/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/)
5. [使用Serilog记录MVC属性](/using-serilog-aspnetcore-in-asp-net-core-3-logging-mvc-propertis-with-serilog/)
6. [在Serilog日志中排除健康检查日志](/using-serilog-aspnetcore-in-asp-net-core-3-excluding-health-check-endpoints-from-serilog-request-logging/)

<!-- more -->

## 当未使用Serilog时的请求日志

使用`dotnet new webapp`创建一个ASP.NET Core 3.0 Razor的应用程序。 其中*Program.cs*，代码如下：

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

在*Startup.cs*的代码如下：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/Error");
        app.UseHsts();
    }

    app.UseHttpsRedirection();
    app.UseStaticFiles();
    app.UseRouting();
    app.UseAuthorization();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapRazorPages();
    });
}
```

启动程序后，会自动访问到应用程序首页。在控制台中，可以看到每次请求都产生了大量的日志，下面只是请求一次首页而产生的日志：

```bash
info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP/2 GET https://localhost:5001/
info: Microsoft.AspNetCore.Routing.EndpointMiddleware[0]
      Executing endpoint '/Index'
info: Microsoft.AspNetCore.Mvc.RazorPages.Infrastructure.PageActionInvoker[3]
      Route matched with {page = "/Index"}. Executing page /Index
info: Microsoft.AspNetCore.Mvc.RazorPages.Infrastructure.PageActionInvoker[101]
      Executing handler method SerilogRequestLogging.Pages.IndexModel.OnGet - ModelState is Valid
info: Microsoft.AspNetCore.Mvc.RazorPages.Infrastructure.PageActionInvoker[102]
      Executed handler method OnGet, returned result .
info: Microsoft.AspNetCore.Mvc.RazorPages.Infrastructure.PageActionInvoker[103]
      Executing an implicit handler method - ModelState is Valid
info: Microsoft.AspNetCore.Mvc.RazorPages.Infrastructure.PageActionInvoker[104]
      Executed an implicit handler method, returned result Microsoft.AspNetCore.Mvc.RazorPages.PageResult.
info: Microsoft.AspNetCore.Mvc.RazorPages.Infrastructure.PageActionInvoker[4]
      Executed page /Index in 221.07510000000002ms
info: Microsoft.AspNetCore.Routing.EndpointMiddleware[1]
      Executed endpoint '/Index'
info: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished in 430.9383ms 200 text/html; charset=utf-8
```

1次请求会产生10条日志。 `Development`环境中，*Microsoft*名称空间下的日志记录级别是“Information”及以上。 如果我们切换到`Production`环境，Microsoft名称空间下的日志记录级别是“Warning”。 现在导航到默认主页会生成以下日志：

```bash

```

啥都没有。上一次运行时，产生的日志都位于Microsoft命名空间下，并且属于“Information”级别，所以它们全部被过滤掉了。 就个人而言，觉得这种处理方式有点过了。如果没有其它日志时，生产环境下最好还是应该输出点日志。

可以通过针对特定的名称空间指定日志级别方式来处理这个问题。例如，调整*Microsoft.AspNetCore.Mvc.RazorPages*名称空间的日志级别为“Warning”，调整*Microsoft*名称空间的日志级别为"Information"，下面是修改后的日志：

```bash
info: Microsoft.AspNetCore.Hosting.Diagnostics[1]
      Request starting HTTP/2 GET https://localhost:5001/
info: Microsoft.AspNetCore.Routing.EndpointMiddleware[0]
      Executing endpoint '/Index'
info: Microsoft.AspNetCore.Routing.EndpointMiddleware[1]
      Executed endpoint '/Index'
info: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished in 184.788ms 200 text/html; charset=utf-8
```

虽然日志中包含了URL、HTTP方法、时间信息、端点等信息。 但是，仍然有让人头疼的地方是生成了4行日志消息。

Serilog的`RequestLoggingMiddleware`中间件可以解决的问题。只创建一条包含所有相关信息的“摘要”日志消息。

## 在应用程序中添加Serilog引用

使用Serilog的`RequestLoggingMiddleware`中间件，只需要添加对Serilog的引用即可。在本节中，将介绍如何添加Serilog到ASP.NET Core的应用程序中。 如果您已经安装了Serilog，请跳至下一部分。

[之前的文章](https://andrewlock.net/adding-serilog-to-the-asp-net-core-generic-host/)介绍了如何将Serilog添加到通用主机应用程序中，当时的ASP.NET Core 2.x已经在通用主机基础架构上进行了重构，因此，在ASP.NET Core 3.0中添加Serilog的方法与与之前非常相似。 本文是参照[Serilog.AspNetCore GitHub](https://github.com/serilog/serilog-aspnetcore)的建议来操作的（同时参考了[Nicholas Blumhardt的帖子](https://nblumhardt.com/2019/10/serilog-in-aspnetcore-3)）。

首先安装*Serilog.AspNetCore NuGet*包，为了方便查看日志，再添加命令行接收器和[Seq](https://datalust.co/seq)接收器。在命令行中执行下面的命令：

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.Seq
```

用Serilog替换默认日志有多种方法。但是推荐在程序一开始就进行替换，比如在`Program.Main`中配置Serilog。 *Program.cs*的代码稍微变得多些：

```csharp
// Additional required namespaces
using Serilog;
using Serilog.Events;

public class Program
{
    public static int Main(string[] args)
    {
        // Create the Serilog logger, and configure the sinks
        Log.Logger = new LoggerConfiguration()
            .MinimumLevel.Debug()
            .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
            .Enrich.FromLogContext()
            .WriteTo.Console()
            .WriteTo.Seq("http://localhost:5341")
            .CreateLogger();

        // Wrap creating and running the host in a try-catch block
        try
        {
            Log.Information("Starting host");
            CreateHostBuilder(args).Build().Run();
            return 0;
        }
        catch (Exception ex)
        {
            Log.Fatal(ex, "Host terminated unexpectedly");
            return 1;
        }
        finally
        {
            Log.CloseAndFlush();
        }
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .UseSerilog() // <- Add this line
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}
```

这样写的好处是，当*appsettings.json*文件缺失、或其内部日志配置缺失时，任然可以输出日志。再启动你的程序，会看到10条日志消息，格式略有差异：

```bash
[13:30:27 INF] Request starting HTTP/2 GET https://localhost:5001/  
[13:30:27 INF] Executing endpoint '/Index'
[13:30:27 INF] Route matched with {page = "/Index"}. Executing page /Index
[13:30:27 INF] Executing handler method SerilogRequestLogging.Pages.IndexModel.OnGet - ModelState is Valid
[13:30:27 INF] Executed handler method OnGet, returned result .
[13:30:27 INF] Executing an implicit handler method - ModelState is Valid
[13:30:27 INF] Executed an implicit handler method, returned result Microsoft.AspNetCore.Mvc.RazorPages.PageResult.
[13:30:27 INF] Executed page /Index in 168.28470000000002ms
[13:30:27 INF] Executed endpoint '/Index'
[13:30:27 INF] Request finished in 297.0663ms 200 text/html; charset=utf-8
```

这样做好像没有解决我们第一节所说的问题，日志数量还是10条，并没有减少。不要慌，因为我们还没有开启 `RequestLoggingMiddleware` 中间件。

## 开启Serilog中间件RequestLoggingMiddleware

`RequestLoggingMiddleware`中间件在*Serilog.AspNetCore*包中，它对每个请求仅生成一条“摘要”日志消息。 添加此中间件很简单。 在您的`Startup`类中，调用`UseSerilogRequestLogging()`即可：

```csharp
// Additional required namespace
using Serilog;

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

注意，ASP.NET Core管道内的中间件顺序很重要。 当请求到达`RequestLoggingMiddleware`时，中间件开始计时，并转交给后续中间件进行处理。 当后续中间件最终生成响应（或引发异常）时，响应通过中间件管道返回到请求记录器，该记录器记录结果并写入摘要日志消息。

只有经过了`RequestLoggingMiddleware`中间件的请求，Serilog才会记录日志。 在上面的示例中，先添加`StaticFilesMiddleware`，再添加`RequestLoggingMiddleware`。 `UseStaticFiles`将请求短路，没有经过后续中间件，直接返回了响应，所以不会被Serilog记录。 通常我们也希望不记录与静态文件相关日志，如果您想记录，只需把`UseSerilogRequestLogging()`移到`UseStaticFiles()`之前。

我们再次运行该应用程序，仍然会看到原始的10条日志消息，同时还一条由Serilog的`RequestLoggingMiddleware`的产生的，倒数第二条消息：

```bash
# Standard logging from ASP.NET Core infrastructure
[14:15:44 INF] Request starting HTTP/2 GET https://localhost:5001/  
[14:15:44 INF] Executing endpoint '/Index'
[14:15:45 INF] Route matched with {page = "/Index"}. Executing page /Index
[14:15:45 INF] Executing handler method SerilogRequestLogging.Pages.IndexModel.OnGet - ModelState is Valid
[14:15:45 INF] Executed handler method OnGet, returned result .
[14:15:45 INF] Executing an implicit handler method - ModelState is Valid
[14:15:45 INF] Executed an implicit handler method, returned result Microsoft.AspNetCore.Mvc.RazorPages.PageResult.
[14:15:45 INF] Executed page /Index in 124.7462ms
[14:15:45 INF] Executed endpoint '/Index'

# Additional Log from Serilog
[14:15:45 INF] HTTP GET / responded 200 in 249.6985 ms

# Standard logging from ASP.NET Core infrastructure
[14:15:45 INF] Request finished in 274.283ms 200 text/html; charset=utf-8
```

关于上面的日志消息，有几点要注意：

- 一条Serilog日志消息就包含您想要的大多数相关信息，HTTP方法、URL路径、状态代码、持续时间。
- Serilog日志记录的持续时间比ASP.NET Core日志记录的持续时间要短。这是正常的，因为Serilog是在进入自己的中间件时才开始计时，直到响应（ response ）被创建并返回后才停止。
- 不管使用ASP.NET Core日志组件，还是使用Serilog日志，附加属性也都会被正常记录到结构化日志中。 例如，RequestId和SpanId（[用于跟踪功能](https://devblogs.microsoft.com/aspnet/improvements-in-net-core-3-0-for-troubleshooting-and-monitoring-distributed-apps/)），都会作为日志附加信息而被记录。 seq的请求的以下图像中看到这一点。
- 默认配置下，会丢失一些信息。 例如，端点名称和Razor页面处理程序被丢失了。 在后续文章中，我将演示如何将它们添加到摘要日志中。

![](https://cdn.ibestread.com/img/seq_request_logging.png)

剩下的工作就是过滤掉“Information”级别的日志消息了。回到*Program.cs*中，调整代码，添加额外的过滤器：

```csharp
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Information)
    // Filter out ASP.NET Core infrastructre logs that are Information and below
    .MinimumLevel.Override("Microsoft.AspNetCore", LogEventLevel.Warning) 
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();
```

现在您的日志会变得既简洁又清晰，每条日志均包含请求的摘要日志信息：

```bash
[14:29:53 INF] HTTP GET / responded 200 in 129.9509 ms
[14:29:56 INF] HTTP GET /Privacy responded 200 in 10.0724 ms
[14:29:57 INF] HTTP GET / responded 200 in 3.3341 ms
[14:30:54 INF] HTTP GET /Missing responded 404 in 16.7119 ms
```

本系列后面的文章会介绍如何在日志中附加额外信息。

## 总结

在本文中，我介绍了如何使用Serilog.AspNetCore的`RequestLoggingMiddleware`中间件，来减少为每次请求生成的日志数量，并同时记录摘要信息。 如果您已经在使用Serilog，则非常容易启用。 只需在*Startup.cs*文件中调用`UseSerilogRequestLogging()`。

当请求到达此中间件时，它将开启计时，当后续的中间件生成响应（或引发异常）时，响应将经过请求记录器后被返回，记录器会记录摘要日志消息。

添加`RequestLoggingMiddleware`中间件之后，可以过滤指定日志级别的日志，并不会丢失有用的信息。