---
title: ASP.Net Core 3.0新特性 启动时的结构化日志
tags: 
  - .NET CORE
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 日志

date: 2020-02-03 10:42:00
---

> 译者:  [Akini Xu](/)
>
> 原文:  [New in ASP.NET Core 3.0: structured logging for startup messages](https://andrewlock.net/new-in-aspnetcore-3-structured-logging-for-startup-messages/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)

此文是 [探索 ASP.NET Core 3.0](/exploring-asp-net-core-3) 第6篇:

1. [`ASP.Net Core 3.0`.csproj文件,Program.cs及通用主机](/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)
2. [`ASP.Net Core 3.0`Startup.cs在不同类型项目中的差异](/comparing-startup-between-the-asp-net-core-3-templates/)
3. [`ASP.Net Core 3.0`新特性-Service provider validation](/new-in-asp-net-core-3-service-provider-validation/)
4. [`ASP.Net Core 3.0`应用程序启动时运行异步任务](/running-async-tasks-on-app-startup-in-asp-net-core-3/)
5. [介绍IHostLifetime及与通用主机间的作用关系](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/)
6. [`ASP.Net Core 3.0`新特性-启动时的结构化日志](/new-in-aspnetcore-3-structured-logging-for-startup-messages/)
7. [`.Net Core 3.0`新特性-本地工具](/new-in-net-core-3-local-tools)

在本文中，我将对ASP.NET Core 3.0应用程序，对启动时记录日志的方式进行一些微小的改动。 现在，ASP.NET Core不再直接将日志输出到控制台中，而是使用日志记录的基础设施组件，生成了结构化日志。

<!-- more --> 

## ASP.NET  Core 2.x中烦人的非结构化日志

当您启动ASP.NET Core 2.x应用程序时，默认情况下，ASP.NET Core会将一些有关您的应用程序的启动信息输出到控制台，例如，当前环境，内容根路径以及Kestrel正在监听的URL： 

```bash
Using launch settings from C:\repos\andrewlock\blog-examples\suppress-console-messages\Properties\launchSettings.json...
Hosting environment: Development
Content root path: C:\repos\andrewlock\blog-examples\suppress-console-messages
Now listening on: https://localhost:5001
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```

这些消息由WebHostBuilder写入的，目的是为您提供了应用程序的启动信息的简单概述，但它直接写入控制台中。 并没有通过*Microsoft.Extensions.Logging*来记录日志，但是除了启动之外的日志又是由*Microsoft.Extensions.Logging*来记录日志。

这样会有两个缺点：

- 这些有用的信息仅写入控制台，因此不会写入其他日志记录器。 
- 写入控制台的消息是非结构化的，它们的格式与其他写入控制台的日志不一样。 甚至没有日志级别或来源。 

最后一点特别烦人，因为在Docker中运行时，通常将日志写入标准输出（控制台），然后让另一个进程读取这些日志并将其发送到日志中心服务器（[例如使用fluentd](https://docs.docker.com/config/containers/logging/fluentd/)）:

![](https://cdn.ibestread.com/img/before_suppression.png)

在ASP.NET Core 2.1中，可以使用环境变量禁用这些日志，[之前文章有介绍](https://andrewlock.net/suppressing-the-startup-and-shutdown-messages-in-asp-net-core/)。 唯一的缺点是启动日志被完全禁用，根本不会记录任何的信息，如图： 

![](https://cdn.ibestread.com/img/after_supression.png)

幸运的是，ASP.NET Core 3.0中的微小改动为我们提供了两全其美的方法！ 

## 在ASP.NET Core 3.0中正确地记录日志

如果使用`dotnet run`启动ASP.NET Core 3.0应用程序，您会看到写入控制台中日志的细微差别： 

```bash
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: https://localhost:5001
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: C:\repos\andrewlock\blog-examples\suppress-console-messages
```

现在，启动日志结构化了！ 这个变化并不是使用`Logger`替代`Console`那么简单。 在ASP.NET Core 2.x中，[由WebHost负责记录启动日志](https://github.com/aspnet/AspNetCore/blob/v2.1.12/src/Hosting/Hosting/src/WebHostExtensions.cs#L83)。 在ASP.NET Core 3.0中，这些消息由`IHostLifetime`记录，在通用主机中此接口默认是由`ConsoleLifetime`实现的，所以说日志是由`ConsoleLifetime`输出的。

在[上一篇文章](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/)中描述了`IhostLifetime`（尤其是`ConsoleLifetime`）的作用，该类负责侦听控制台中的`Ctrl + C`按键，并执行关闭过程。 

`ConsoleLifetime`在`WaitForStartAsync()`方法中绑定了2个事件的回调方法，当触发`ApplicationLifetime.ApplicationStarted`事件或`ApplicationLifetime.ApplicationStopping`事件时，会调用对应的方法：

```csharp
public Task WaitForStartAsync(CancellationToken cancellationToken)
{
    if (!Options.SuppressStatusMessages)
    {
        // Register the callbacks for ApplicationStarted
        _applicationStartedRegistration = ApplicationLifetime.ApplicationStarted.Register(state =>
        {
            ((ConsoleLifetime)state).OnApplicationStarted();
        },
        this);

        // Register the callbacks for ApplicationStopping
        _applicationStoppingRegistration = ApplicationLifetime.ApplicationStopping.Register(state =>
        {
            ((ConsoleLifetime)state).OnApplicationStopping();
        },
        this);
    }

    // ...

    return Task.CompletedTask;
}
```

对应的回调函数`OnApplicationStarted（）`，`OnApplicationStarted（）`中，使用日志组件记录了启动或关闭信息： 

```cs
private void OnApplicationStarted()
{
    Logger.LogInformation("Application started. Press Ctrl+C to shut down.");
    Logger.LogInformation("Hosting environment: {envName}", Environment.EnvironmentName);
    Logger.LogInformation("Content root path: {contentRoot}", Environment.ContentRootPath);
}

private void OnApplicationStopping()
{
    Logger.LogInformation("Application is shutting down...");
}
```

在`Systed Lifetime`和`Windows ServiceLifetime`下可能略有不同，但实现原理还是一样的。

## 使用ConsoleLifetime屏蔽启动日志

当使用`ConsoleLifetime`记录的启动日志时，发现一个令人惊讶的变化：无法再按照我[之前文章](https://andrewlock.net/suppressing-the-startup-and-shutdown-messages-in-asp-net-core/)中的方式屏蔽启动日志了。 无论是否设置了环境变量`ASPNETCORE_SUPPRESSSTATUSMESSAGES`都没有任何效果，日志都会被记录！ 

上一节已经说过，由于使用了*Microsoft.Extensions.Logging*来记录日志，那这个不是什么大问题。 如果你不想看到这些启动日志，则可以在`Startup.cs`中配置`ConsoleLifetimeOptions`： 

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // ... other configuration
        services.Configure<ConsoleLifetimeOptions>(opts => opts.SuppressStatusMessages = true);
    }
}
```

现在只显示结构化的日志信息了。如果需要，还可以在环境变量`ASPNETCORE_SUPPRESSSTATUSMESSAGES` 中设置是否开启屏蔽：

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }

    public Startup(IConfiguration configuration) => Configuration = configuration;

    public void ConfigureServices(IServiceCollection services)
    {
        // ... other configuration
        services.Configure<ConsoleLifetimeOptions>(opts 
                => opts.SuppressStatusMessages = Configuration["SuppressStatusMessages"] != null);
    }
}
```

另外，就算屏蔽了启动日志，`Kestrel`  任然会输出监听的URLs，这个是没有办法屏蔽的：

```bash
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: http://0.0.0.0:5000
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: https://0.0.0.0:5001
```

## 总结

在本文中，说明了如何在3.0中输出结构化日志，避免像ASP.NET Core 2.x应用程序那样，输出烦人的启动时非结构化日志。 这样可以确保将日志写入所有已配置的记录器，并且在控制台输出标准格式。 另外，还说明了如何配置`ConsoleLifetimeOptions`来屏蔽启动日志。 

