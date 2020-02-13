---
title: 使用Serilog减少日志的详细程度
tags: 
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 日志

date: 2020-02-13 07:30:00
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

Serilog的`RequestLoggingMiddleware`中间件就是为了解决的问题。只创建一条包含所有相关信息的“摘要”日志消息。

## 在应用程序中添加Serilog



使用Serilog的RequestLoggingMiddleware的一个依赖项是您正在使用Serilog！ 在本节中，我将介绍将Serilog添加到ASP.NET Core应用程序的基础。 如果您已经安装了Serilog，请跳至下一部分。