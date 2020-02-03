---
title: 介绍IHostLifetime及与通用主机间的作用关系
tags: 
  - dotnet-core
  - asp-dotnet-core
  - asp-dotnet-core-3
  - IHostLifetime

date: 2020-02-02 10:49:00
---

> 译者:  [Akini Xu](/)
>
> 原文:  [Introducing IHostLifetime and untangling the Generic Host startup interactions](https://andrewlock.net/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [探索 ASP.NET Core 3.0](/exploring-asp-net-core-3) 第5篇:

1. [ASP.Net Core 3.0中的.csproj文件,Program.cs及通用主机](/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)
2. [ASP.Net Core 3.0的Startup.cs在不同项目类型中的差异](/comparing-startup-between-the-asp-net-core-3-templates/)
3. [ASP.Net Core 3.0的新特性-Service provider validation](/new-in-asp-net-core-3-service-provider-validation/)
4. [ASP.Net Core 3.0中应用程序启动时运行异步任务](/running-async-tasks-on-app-startup-in-asp-net-core-3/)
5. 介绍IHostLifetime及与通用主机间的作用关系(本文)
6. [ASP.Net Core 3.0的新特性-启动时的结构化日志](/new-in-aspnetcore-3-structured-logging-for-startup-messages/)
7. [ASP.Net Core 3.0的新特性-本地工具](/new-in-net-core-3-local-tools)

在本文中，将介绍如何在通用主机上重新构建ASP.NET Core 3.0，以及由此带来的一些好处。 另外还展示了3.0中引入的新的抽象类`IHostLifetime`，并介绍它在管理应用程序（尤其是` worker services `）生命周期中的作用。

在文章的后半部分，我会详细介绍各个类之间的如何交互，及它们在应用程序启动和关闭期间的作用。 同时也会详细介绍通常不需要我们处理的事情，即使不需要关心，但是理解其原理对于我们也很有必要！ 

<!-- more --> 

## 背景 在通用主机上重新构建ASP.NET Core 3.0

ASP.NET Core 3.0的主要功能之一就是整体框架都已基于[.NET 通用主机](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.0)进行了重写。 .NET 通用主机是在ASP.NET Core 2.1中引入的，它是ASP.NET Core`WebHost`的“非Web”版本。通用主机允许您在非Web情况下使用*Microsoft.Extensions*中的功能，比如依赖注入，配置和日志记录等。 

这绝对是一个令人羡慕的功能，但是实际使用中也存在一些问题。通用主机本质上直接复制了ASP.NET Core所需的许多抽象类，但是使用不同的名称空间。例如`IHostingEnvironment`，在1.0版本中位于*Microsoft.AspNetCore.Hosting*，但2.1版本中，又在*Microsoft.Extensions.Hosting*名称空间下添加了新的`IHostingEnvironment`。即使接口内容是相同的，也会导致[针对此接口的扩展方法冲突](https://andrewlock.net/the-asp-net-core-generic-host-namespace-clashes-and-extension-methods/)。

在3.0中，ASP.NET Core团队进行较大的重构，直接解决此问题。他们在通用主机的架构上重写了ASP.NET Core主机，来替代里两个独立的主机。 这意味着它可以真正重用相同的抽象，从而解决了上述问题。 此举动机是希望在通用主机之上构建其他非HTTP协议的主机（例如ASP.NET Core 3.0中引入的[gRPC](https://docs.microsoft.com/en-us/aspnet/core/tutorials/grpc/grpc-start?view=aspnetcore-3.0&tabs=visual-studio)功能）。

在ASP.NET Core 3中，对通用主机之上进行“重构”或“重新平台化”的真正意义是什么？ 从根本上讲，这意味着Kestrel Web服务器（处理HTTP请求和对中间件管道的调用）只是作为`IHostedService`运行。 我已经在博客上写了很多有关创建` Hosted Service `的文章，现在应用程序启动时，`Kestrel`只是在后台运行的另一个服务而已。

>  注意：值得强调的一点是，在ASP.NET Core 2.x应用程序中使用的已有`WebHost`和`WebHostBuilder`的相关实现在3.0中并没有消失。 它们不再被**推荐**，也没有被删除，甚至没有被标记为过时。 我希望它们会在下一个主要版本中被标记为过时，因此值得考虑进行切换。 

上面简单介绍了背景。 在通用主机中，Kestrel是作为`IHostedService`运行的。 另外，ASP.NET Core 3.0中引入的另一个功能是`IHostLifetime`接口，该接口允许使用其它托管模型。 

## Worker services和新的IHostLifetime接口

在ASP.NET Core 3.0引入了“[worker services](https://devblogs.microsoft.com/aspnet/net-core-workers-as-windows-services/)”的概念以及相关的新应用程序模板。*Worker services*旨在提供可长时间运行的应用程序，可以将它们安装为*Windows Service*或*Systemd service*。 这些服务有两个主要功能： 

- 通过实现` IHostedService `接口，来实现后台服务应用程序。
- 通过实现` IHostLifetime `接口，来管理后台服务应用程序的生命周期。

第一点中的`IHostedService`已经存在了很长时间，并且允许您运行后台服务。 第二点则是有趣的地方，`IHostLifetime`接口是.NET Core 3.0的新增功能，它具有[两种方法](https://github.com/aspnet/Extensions/blob/d656c4f7e22d1c0b84cab1b453c50ce73c89a071/src/Hosting/Abstractions/src/IHostLifetime.cs)：

```csharp
public interface IHostLifetime
{
    Task WaitForStartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

我们先总体说明下这2方法，在稍后的部分中，再将详细介绍：

- ` WaitForStartAsync `：在通用主机启动时（**starting**）被调用，可用于启动侦听关闭事件或延迟应用程序的启动，直到发生某些事件为止。
-  `StopAsync` ：在通用主机停止时（**stopping**）被调用。 

目前在.NET Core 3.0三个不同`IHostLifetime`实现：

-  [`ConsoleLifetime`](https://github.com/aspnet/Extensions/blob/2875a1ad56073d98cf0856fa72620d6fdfca588d/src/Hosting/Hosting/src/Internal/ConsoleLifetime.cs) ： 侦听`SIGTERM`或`Ctrl + C`并停止主机应用程序
-  [`SystemdLifetime`](https://github.com/aspnet/Extensions/blob/531db4e313e577192cdbca3931c4298b6db1e611/src/Hosting/Systemd/src/SystemdLifetime.cs) ： 监听`SIGTERM`并停止主机应用程序，并通知`systemd`状态变化（`Ready`和`Stopping`） 
-  [`WindowsServiceLifetime`](https://github.com/aspnet/Extensions/blob/14c02660827dab4706b3db0a938d923d24fa6d0f/src/Hosting/WindowsServices/src/WindowsServiceLifetime.cs) ： 挂接到Windows服务事件生命周期管理 

在 ASP.NET Core 2.x应用程序中，通用主机默认使用`ConsoleLifetime`，当应用程序从控制台接收到`SIGTERM`信号或`Ctrl + C`时，应用程序将停止。如果是创建*Worker services*（Windows服务或systemd服务）时，则需要配置`IHostLifetime`。

## 理解应用程序启动

当我在研究这个新的抽象时，开始感到非常困惑。它什么时候会被调用？ 它与`ApplicationLifetime`有什么关系？ 谁先调用了`IHostLifetime`？ 为了使事情更清晰，我花了一些时间，在ASP.NET Core 3.0的默认应用程序中找出他们之间的调用关系。 

我们从ASP.NET Core 3.0的`Program.cs`文件开始分析，[本系列第一篇文章中有写道](exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)： 

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

我感兴趣的是当创建了通用主机对象后，`run()`的方法到底做了些什么？

> 请注意，我不会分析所有的代码-我会跳过无关紧要的内容。 我的目标是对调用的过程有一个整体了解。如果您想更深入一点，可以 查看[源代码](https://github.com/aspnet/AspNetCore)！ 

`Run()`是[HostingAbstractionsHostExtensions的扩展方法](https://github.com/aspnet/Extensions/blob/494e2c53cd/src/Hosting/Abstractions/src/HostingAbstractionsHostExtensions.cs#L49)，它调用了异步`RunAsync()`方法，并阻塞直到该方法退出。 当该方法退出时，应用程序也会退出。 下图是`RunAsync()`执行的时序图，后面将讨论详细过程： 

![](https://cdn.ibestread.com/img/program_startup.svg)

*Program.cs*调用`Run()`扩展方法，该方法调用`RunAsync()`扩展方法。 然后再调用`IHost`实例上的`StartAsync()`。 `StartAsync`方法会完成一些其他工作，例如，启动`IHostingServices`（稍后将介绍），该方法在被调用后会快速返回。 

接下来，`RunAsync()`方法会调用另一个扩展方法`WaitForShutdownAsync()`。 此扩展方法的作用与它的方法名是一致的（等待关闭）。 此方法先对其自身进行配置，以使其暂停，直到`IHostApplicationLifetime`的`ApplicationStopping Token `触发为止（等会我们再讲解如何触发该`Token`）。 

扩展方法`WaitForShutdownAsync()`，使用`TaskCompletionSource`，并等待关联的`Task`来实现此目的。它看起来很有趣，这并不是我以前使用过的模式，源码如下：（[HostingAbstractionsHostExtensions](https://github.com/aspnet/Extensions/blob/494e2c53cd/src/Hosting/Abstractions/src/HostingAbstractionsHostExtensions.cs#L86-L107)） 

```csharp
public static async Task WaitForShutdownAsync(this IHost host)
{
    // Get the lifetime object from the DI container
    var applicationLifetime = host.Services.GetService<IHostApplicationLifetime>();

    // Create a new TaskCompletionSource called waitForStop
    var waitForStop = new TaskCompletionSource<object>(TaskCreationOptions.RunContinuationsAsynchronously);

    // Register a callback with the ApplicationStopping cancellation token
    applicationLifetime.ApplicationStopping.Register(obj =>
    {
        var tcs = (TaskCompletionSource<object>)obj;

        // When the application stopping event is fired, set 
        // the result for the waitForStop task, completing it
        tcs.TrySetResult(null);
    }, waitForStop);

    // Await the Task. This will block until ApplicationStopping is triggered,
    // and TrySetResult(null) is called
    await waitForStop.Task;

    // We're shutting down, so call StopAsync on IHost
    await host.StopAsync();
}
```

此扩展方法解释了如何“暂停”应用程序的运行状态，而让所有任务都在后台任务中运行。 让我们更深入地了解上图顶部的`IHost.StartAsync()`方法调用。 让我们更深入的了解上图中` IHost.StartAsync() `方法的调用。

## Host.StartAsync()

在上图中，我们研究了在接口`IHost`上运行的`HostingAbstractionsHostExtensions`扩展方法。 如果我们想知道在调用`IHost.StartAsync()`时通常会发生什么，那么我们需要看看具体的实现。 下图显示了[通用主机是如何实现的StartAsync()](https://github.com/aspnet/Extensions/blob/494e2c53cd/src/Hosting/Hosting/src/Internal/Host.cs#L35)方法：

![](https://cdn.ibestread.com/img/hosting_startup.svg)

从上图可以看到，其中还是有很多步骤的！ 在`Host.StartAsync()`方法中，首先调用了`IHostLifetime`实例的上`WaitForStartAsync()`方法。 `Host.StartAsync()`具体实现取决于您使用的是哪个`IHostLifetime`，假定我们正在使用`ConsoleLifetime`（ASP.NET Core应用程序的默认设置）。

> **注意**：[`SystemdLifetime`](https://github.com/aspnet/Extensions/blob/494e2c53cd/src/Hosting/Systemd/src/SystemdLifetime.cs)的行为与`ConsoleLifetime`非常相似，并具有一些额外的功能。 [`WindowsServiceLifetime`](https://github.com/aspnet/Extensions/blob/494e2c53cd/src/Hosting/WindowsServices/src/WindowsServiceLifetime.cs)则完全不同的，它派生于`System.ServiceProcess.ServiceBase`。 

[`ConsoleLifetime.WaitForStartAsync()`方法](https://github.com/aspnet/Extensions/blob/494e2c53cd/src/Hosting/Hosting/src/Internal/ConsoleLifetime.cs#L44)（如下所示）做了一件重要的事情：它为控制台中的`SIGTERM`请求和`Ctrl + C`添加了事件侦听器。 当准备关闭应用程序时将触发这些事件。 因此，通常由`IHostLifetime`负责控制应用程序何时关闭。

```csharp
public Task WaitForStartAsync(CancellationToken cancellationToken)
{
    // ... logging removed for brevity

    // Attach event handlers for SIGTERM and Ctrl+C
    AppDomain.CurrentDomain.ProcessExit += OnProcessExit;
    Console.CancelKeyPress += OnCancelKeyPress;

    // Console applications start immediately.
    return Task.CompletedTask;
}
```

如上面的代码所示，此方法立即完成，并将控制权返回给`Host.StartAsync()`。 此时，主机加载所有`IHostedService`实例，并调用每个实例的`StartAsync()`方法。还包括用于[启动`Kestrel Web`服务器](https://github.com/aspnet/AspNetCore/blob/v3.0.0-rc1.19457.4/src/Hosting/Hosting/src/GenericHost/GenericWebHostedService.cs)的`GenericWebHostService`（该`Hosted Service`最后启动，上一篇曾提到过的[应用程序启动时运行异步任务](/running-async-tasks-on-app-startup-in-asp-net-core-3)）。

一旦所有`IHostedServices`全部启动，`Host.StartAsync()`就会调用`IHostApplicationLifetime.NotifyStarted()`方法，来触发所有已绑定的回调方法（通常只是记录）并退出。

> **请注意**，`IHostLifetime`与`IHostApplicationLifetime`是**不同**的。 前者用于控制应用程序何时启动。 后者（由[ApplicationLifetime实现](https://github.com/aspnet/Extensions/blob/494e2c53cd/src/Hosting/Hosting/src/Internal/ApplicationLifetime.cs)）包含`CancellationTokens`，用于应用程序各个生命周期的事件回调绑定。

此时，应用程序处于“运行”状态，所有后台服务都在运行中、Kestrel处理请求、扩展方法`WaitForShutdownAsync()`等待`ApplicationStopping`事件的触发。 最后，我们在控制台中键入`Ctrl + C`，看看会发生什么？

## shotdown process

当`ConsoleLifetime`从控制台接收到`SIGTERM`信号或`Ctrl + C`时，将引发关闭过程。 下图显示了关闭过程中所有关键参与者之间的相互作用： 

![](https://cdn.ibestread.com/img/hosting_shutdown.svg)

- 当触发`Ctrl + C`终止事件时，`ConsoleLifetime`会调用`IHostApplicationLifetime.StopApplication()`方法。 它将触发所有绑定了`ApplicationStopping`取消令牌的回调。 本文开头写的[理解应用程序启动](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/#理解应用程序启动)，`RunAsync()`正在等待着`ApplicationStopping`的触发，当`await`的任务完成时，调用了`Host.StopAsync()`。

- `Host.StopAsync()`方法中第二次调用`IHostApplicationLifetime.StopApplication()`方法。 这次调用是个空方法，从技术上讲，这是必需的，因为还有其他方式，导致`Host.StopAsync()`被触发（本文是从`Ctrl + C`触发）。

- 接下来，主机以相反的顺序关闭所有`IHostedServices`。 最先启动的服务将被最后关闭，因此`GenericWebHostedService`第一个被关闭。

- 服务关闭后，将调用`IHostLifetime.StopAsync`，对于`ConsoleLifetime`和`SystemdLifetime`来说，都是空操作，但对于`WindowsServiceLifetime`来说，有其他逻辑要执行的。 最后，`Host.StopAsync()`在退出之前，调用`IHostApplicationLifetime.NotifyStopped()`以通知其它关联的处理程序。

- 此时，所有的都关闭，`Program.Main`函数退出，应用程序退出。

## 总结

在这篇文章中，我们了解一些有关如何在通用主机之上重新构建ASP.NET Core 3.0的背景知识，并介绍了新的`IHostLifetime`接口。 然后，我详细讲述了，通用主机上的ASP.NET Core 3.0应用程序在启动和关闭时，各个类之间的如何交互及它们的作用。

显然这需要一个漫长的过程， 我个人认为通过查看代码，可以让你理解的更加深刻，希望本文也可以对其他人有所帮助！