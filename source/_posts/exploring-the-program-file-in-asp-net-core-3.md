---
title: ASP.Net Core 3.0中的Program.cs文件
tags: 
  - dotnet core
  - asp dotnet core
  - asp dotnet core 3.0
  - Program

categories:
  - 系列
  - 探索ASP.NETCore3.0

date: 2020-01-31 10:31:57
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Exploring the new project file, Program.cs, and the generic host](https://andrewlock.net/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [探索 ASP.NET Core 3.0](https://blog.ibestread.com/exploring-asp-net-core-3) 第2篇:

1. [ASP.Net Core 3.0中的.csproj文件](https://blog.ibestread.com/exploring-the-new-project-file-in-asp-net-core-3/)
2. ASP.Net Core 3.0中的Program.cs文件(本文)
3. [ASP.Net Core 3.0中的通用主机](https://blog.ibestread.com/exploring-the-generic-host-in-asp-net-core-3/)
4. [ASP.Net Core 3.0的Startup.cs在不同项目类型中的差异](https://blog.ibestread.com/comparing-startup-between-the-asp-net-core-3-templates/)
5. [ASP.Net Core 3.0的新特性-Service provider validation](https://blog.ibestread.com/new-in-asp-net-core-3-service-provider-validation)
6. [ASP.Net Core 3.0中应用程序启动时运行异步任务](https://blog.ibestread.com/running-async-tasks-on-app-startup-in-asp-net-core-3)
7. [介绍IHostLifetime及与通用主机间的作用关系](https://blog.ibestread.com/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions)
8. [ASP.Net Core 3.0的新特性-启动时的结构化日志](https://blog.ibestread.com/new-in-aspnetcore-3-structured-logging-for-startup-messages)
9. [ASP.Net Core 3.0的新特性-本地工具](https://blog.ibestread.com/new-in-net-core-3-local-tools)

在这篇文章中我们来看看`ASP.NET Core 3.0`的应用程序的基础组件 : `.csproj`的项目文件和`Program.cs`启动文件。我将介绍`ASP.NET Core 3.0`与`ASP.NET Core 2.X`的差异，并讨论相关APIs的使用变化。 

 <!-- more --> 

## Program.cs 在 2.X和3.0中的变化

在ASP.NET Core 3.0中的*Program.cs*的代码看上去和ASP.NET Core 2.X非常相似，但是实际上很多类型都发生了变化。这是因为在.NET Core 3.0中，ASP.NET Core已经重构，使用[通用主机（Generic Host）](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.0)替代[Web主机（Web Host）](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host?view=aspnetcore-3.0)。 

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

>通用主机（Generic Host）是在2.1引入，是一个不错的主意，但我发现它有[各种问题](https://andrewlock.net/the-asp-net-core-generic-host-namespace-clashes-and-extension-methods/)，主要原因是使用其他类库时会[增添额外的工作](https://andrewlock.net/adding-serilog-to-the-asp-net-core-generic-host/)。值得庆幸的是在3.0中解决这些问题。 

在大多数情况下，与在.NET Core 2.x中使用的最终结果非常相似，它分为两个步骤。 除了使用单个方法`WebHost.CreateDefaultBuilder()`来配置应用程序的之外，还有另外两个单独的方法调用： 

- ` Host.CreateDefaultBuilder() ` 。配置应用配置，日志记录和依赖注入容器。 
- ` IHostBuilder.ConfigureWebHostDefaults() `。 为默认的ASP.NET Core应用程序添加基础设施，包括配置` Kestrel `，使用` Startup.cs `来配置依赖注入容器和中间件。
