---
title: ASP.Net Core 3.0 应用程序启动时运行异步任务
tags: 
  - .NET CORE
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 异步
  - 后台任务

date: 2020-02-01 19:03:00
---

> 译者:  [Akini Xu](/)
>
> 原文:  [Running async tasks on app startup in ASP.NET Core 3.0](https://andrewlock.net/running-async-tasks-on-app-startup-in-asp-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [探索 ASP.NET Core 3.0](/exploring-asp-net-core-3) 第4篇:

1. [`ASP.Net Core 3.0`.csproj文件,Program.cs及通用主机](/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)
2. [`ASP.Net Core 3.0`Startup.cs在不同类型项目中的差异](/comparing-startup-between-the-asp-net-core-3-templates/)
3. [`ASP.Net Core 3.0`新特性-Service provider validation](/new-in-asp-net-core-3-service-provider-validation/)
4. [`ASP.Net Core 3.0`应用程序启动时运行异步任务](/running-async-tasks-on-app-startup-in-asp-net-core-3/)
5. [介绍IHostLifetime及与通用主机间的作用关系](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/)
6. [`ASP.Net Core 3.0`新特性-启动时的结构化日志](/new-in-aspnetcore-3-structured-logging-for-startup-messages/)
7. [`.Net Core 3.0`新特性-本地工具](/new-in-net-core-3-local-tools)

在本文中，我将介绍如何在ASP.NET Core 3.0的`WebHost`中做微小的改动，就能让应用程序在启动时，更方便地运行异步任务。

 <!-- more --> 

## 在程序启动时运行异步任务

在之前一个系列-[应用程序启动时运行异步任务的多种方式](https://andrewlock.net/series/running-async-tasks-on-app-startup-in-asp-net-core/)中有过说明。 你有很多原因需要在程序启动时执行一些异步任务，例如，运行数据库迁移，[验证强类型配置](https://andrewlock.net/adding-validation-to-strongly-typed-configuration-objects-in-asp-net-core/)或填充缓存。

不幸的是，在2.x中，无法使用任何内置的ASP.NET Core语法来实现此目的：

- `IStartupFilter`只有同步API，因此只能通过[同步调用异步方法](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md#asynchrony-is-viral)实现。
- `IApplicationLifetime`只有同步API，并在服务器开始处理请求后引发`ApplicationStarted`事件。
- `IHostedService`具有异步API，但只在服务器启动并开始处理请求后才执行。

对此，我提出了两种可能的解决方案：

- 在 `WebHost Build`之后 ，在`WebHost Run`之前，[手动执行任务](https://andrewlock.net/running-async-tasks-on-app-startup-in-asp-net-core-part-2/#an-example-async-database-migration)。
- 在服务器启动后，但接收请求之前，使用[自定义`IServer`实现](https://andrewlock.net/running-async-tasks-on-app-startup-in-asp-net-core-part-2/#an-alternative-approach-by-decorating-iserver)运行任务。 不幸的是，这种方法可能会有[问题](https://github.com/andrewlock/NetEscapades.AspNetCore.StartupTasks/issues/4)。

在ASP.NET Core 3.0中，我们仅仅对`WebHost`代码进行微小的改动，就将带来很大的不同。我们不再需要上面那些解决方案，并且可以放心地使用`IHostedService`。

## 小改动 大不同

在ASP.NET Core 2.x中，可以通过实现`IHostedService`来[运行后台服务](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-2.2&tabs=visual-studio)。 在应用程序开始处理请求后不久（即，在Kestrel Web服务器启动之后）启动，并在应用程序关闭时停止。

在ASP.NET Core 3.0中，`IHostedService`仍具有相同的目的-运行后台任务。 但是由于WebHost的微小更改，现在还可以使用它在应用程序启动时，自动运行异步任务。

在ASP.NET Core 2.x中，[`WebHost`的代码如下](https://github.com/aspnet/AspNetCore/blob/v2.1.12/src/Hosting/Hosting/src/Internal/WebHost.cs#L153)：

```cs
public class WebHost
{
    public virtual async Task StartAsync(CancellationToken cancellationToken = default)
    {
        // ... initial setup
        await Server.StartAsync(hostingApp, cancellationToken).ConfigureAwait(false);

        // Fire IApplicationLifetime.Started
        _applicationLifetime?.NotifyStarted();

        // Fire IHostedService.Start
        await _hostedServiceExecutor.StartAsync(cancellationToken).ConfigureAwait(false);

        // ...remaining setup
    }
}
```

在ASP.NET Core 3.0中，[进行了如下调整](https://github.com/aspnet/AspNetCore/blob/v3.0.0-preview9.19424.4/src/Hosting/Hosting/src/Internal/WebHost.cs#L154)：

```csharp
public class WebHost
{
    public virtual async Task StartAsync(CancellationToken cancellationToken = default)
    {
        // ... initial setup

        // Fire IHostedService.Start
        await _hostedServiceExecutor.StartAsync(cancellationToken).ConfigureAwait(false);

        // ... more setup
        await Server.StartAsync(hostingApp, cancellationToken).ConfigureAwait(false);

        // Fire IApplicationLifetime.Started
        _applicationLifetime?.NotifyStarted();

        // ...remaining setup
    }
}
```

如您所见，`IHostedService.Start`现在位于`Server.StartAsync`之前执行。 这个更改意味着您现在可以使用`IHostedService`运行异步任务。

## 在程序启动时使用IHostedService异步任务

通过实现一个`IHostedService`，来作为应用程序启动任务并不是件难事。该接口包含了两个方法： 

```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

需要在接收请求之前运行的任何代码，都应放在`StartAsync`方法中。 在这种情况下，可以忽略StopAsync方法。

例如，下面代码是在应用程序启动时，运行EF Core的迁移任务： 

```csharp
public class MigratorHostedService: IHostedService
{
    // We need to inject the IServiceProvider so we can create 
    // the scoped service, MyDbContext
    private readonly IServiceProvider _serviceProvider;
    public MigratorHostedService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        // Create a new scope to retrieve scoped services
        using(var scope = _serviceProvider.CreateScope())
        {
            // Get the DbContext instance
            var myDbContext = scope.ServiceProvider.GetRequiredService<MyDbContext>();

            //Do the migration asynchronously
            await myDbContext.Database.MigrateAsync();
        }
    }

    // noop
    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

把将要运行的任务添加到依赖注入容器中，并使其在应用开始接收请求之前运行，使用`AddHostedService<>()`扩展方法：

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        // other DI configuration
        services.AddHostedService<MigratorHostedService>();
    }

    public void Configure(IApplicationBuilder)
    {
        // ...middleware configuration
    }
}
```

被添加的后台服务会按照添加到容器中的顺序被依次执行。

## 总结

在本文中，介绍如何在ASP.NET Core 3.0的`WebHost`中做微小的改动，就能让应用程序在启动时，更方便地运行异步任务。在ASP.NET Core 2.x中，并没有一个完美的解决办法。3.0的更改意味着可以使用`IHostedService`来运行异步任务。