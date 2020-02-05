---
title: ASP.Net Core 3.0 .csproj文件,Program.cs及通用主机
tags: 
  - .NET CORE
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 项目文件
  - 通用主机

date: 2020-01-31 10:31:58
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Exploring the new project file, Program.cs, and the generic host](https://andrewlock.net/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [探索 ASP.NET Core 3.0](/exploring-asp-net-core-3) 第1篇:

1. [`ASP.Net Core 3.0`.csproj文件,Program.cs及通用主机](/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)
2. [`ASP.Net Core 3.0`Startup.cs在不同类型项目中的差异](/comparing-startup-between-the-asp-net-core-3-templates/)
3. [`ASP.Net Core 3.0`新特性-Service provider validation](/new-in-asp-net-core-3-service-provider-validation/)
4. [`ASP.Net Core 3.0`应用程序启动时运行异步任务](/running-async-tasks-on-app-startup-in-asp-net-core-3/)
5. [介绍IHostLifetime及与通用主机间的作用关系](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/)
6. [`ASP.Net Core 3.0`新特性-启动时的结构化日志](/new-in-aspnetcore-3-structured-logging-for-startup-messages/)
7. [`.Net Core 3.0`新特性-本地工具](/new-in-net-core-3-local-tools)

在这篇文章中我们来看看`ASP.NET Core 3.0`的应用程序的基础组件 : `.csproj`的项目文件和`Program.cs`启动文件。我将介绍`ASP.NET Core 3.0`与`ASP.NET Core 2.X`的差异，并讨论相关APIs的使用变化。 

<!-- more --> 

## 介绍

`.NET Core 3.0`的核心是预计于9月23日的 [.NET Conf](https://www.dotnetconf.net/) 期间发布，~~现在预览版（`Preview 8`）现已发布~~现在.NET Core 3.1 已发布。最终` Release版 `与`Preview版`不会有太多的差异，我们现在来看看到底带来了些什么。 

`.NET Core 3.0`的主要功能集中在Windows桌面应用程序可以在`.NET Core`上运行，另外`ASP.NET Core`也有很多新功能。其中最大的新功能是 [server-side Blazor](https://docs.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-3.0#server-side) （~~我更感兴趣的客户端版本个人，这是不完全产品尚未推出~~目前已经更新）。 

在这篇文章中,我们着重关注下面2个基础设施的变化:

- Nuget中的 [Microsoft.AspNetCore.App metapackage](https://andrewlock.net/the-microsoft-aspnetcore-all-metapackage-is-huge-and-thats-awesome-thanks-to-the-net-core-runtime-store-2/) 不再可用.
- `ASP.NET Core` 使用通用主机( [Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.0) )替代了Web主机([Web Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host?view=aspnetcore-3.0) )。

>  如果你准备升级ASP.NET Core 2.x应用程序到3.0，需要仔细查看微软的官方迁移指导( [the migration guidance on docs.microsoft.com](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30) )。 

在这篇文章中，我们创建一个`ASP.NET Core 3.0`的应用程序（例如：使用` dotnet new webapi `），并来看看`.csproj`文件，`Program.cs`中的变化。**在后面的文章中，我们会使用不同的项目模板（`Web`，`webapi`，`mvc`等）创建项目，并比较`Startup.cs`文件的变化差异。** 

## 新的项目文件及目标框架

创建一个新的ASP.NET Core项目文件后，编辑`.csproj`文件，代码如下：

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.0</TargetFramework>
  </PropertyGroup>

</Project>
```

与`ASP.NET Core 2.X`对比后，发现有如下相同及差异：

- `<Project>`中的`Sdk`节点依然是` Microsoft.Net.Sdk.Web `。在新旧的项目文件中2者是一样的。
- `<TargetFramework>` 由` netcoreapp2.1 `或` netcoreapp2.2 `改为` netcoreapp3.0 `。原因是新版本依赖`.Net Core 3.0`。
-  无需再对`Microsoft.AspNetCore.App`元包进行引用。

最有趣变化是最后一点。在ASP.NET Core 2.1/ 2.2，你可以引用*shared framework*的元数据包*Microsoft.AspNetCore.App*，正如我在以前的文章[shared framework](https://andrewlock.net/exploring-the-microsoft-aspnetcore-app-shared-framework-in-asp-net-core-2-1-preview-1/)中描述。*shared framework*提供了许多好处，例如避免在应用程序手动安装的所有单独的软件包，并允许运行时的向前更新。 

在ASP.NET 3.0 Core中，微软将不再发布*shared framework*作为的NuGet元数据包。没有了*Microsoft.AspNetCore.App 3.0.0*版本。*shared framework*的安装仍与.NET Core以前一样，但你在不同的3.0引用它。 

在ASP.NET Core 2.X中，为了引用*shared framework*，需要以下内容添加到项目文件： 

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.App" />
</ItemGroup>
```

在ASP.NET Core 3.0中，需要使用`<FrameworkReference>` ：

```xml
<ItemGroup>
  <FrameworkReference Include="Microsoft.AspNetCore.App" />
</ItemGroup>
```

你可能会问："为什么我的ASP.NET Core项目文件中没有这个？" 。 

这个问题问的很好 。因为，SDK `Microsoft.Net.Sdk.Web`默认情况包括了 `Microsoft.AspNetCore.App`。

## 不要在shared framework组件中添加额外的package

在ASP.NET Core 3.0中另外一个最大的变化就是[不要在shared framework组件中添加额外的package](https://github.com/aspnet/AspNetCore/issues/3756)。例如：在ASP.NET Core 2.X中需要单独引用 *Microsoft.AspNetCore.Authentication*  或  *Microsoft.AspNetCore.Identity* 2个packages。

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.Authentication" Version="2.1.0"/>
  <PackageReference Include="Microsoft.AspNetCore.Identity" Version="2.1.0"/>
</ItemGroup>
```

我们通常会这样来引用一些类库或packages。但是在应用程序中总是会依赖一些必要的*shared framework*。在ASP.NET 3.0 Core中这样就行不通了，因为这些packages将不会在`Nuget`中发布了。如果你想引用这些packages则需要在`<FrameworkReference>` 中添加。

> 另外值得注意的一件事，有一些packages，例如 EF Core等，都[不再属于*shared framework*](https://github.com/aspnet/AspNetCore/issues/3755)。如果你想引用，必须手动在`PackageReference`中添加。
>
> 查看完整列表，请访问  [see this GitHub issue](https://github.com/aspnet/AspNetCore/issues/3755) 

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

## 通用主机构造器

正如我已经提到的，通用主机是构建*ASP.NET Core 3.0*应用程序的基础。 它提供了在ASP.NET Core应用程序中惯用的基础`Microsoft.Extensions.*`元素，例如日志记录，配置和依赖项注入。

下面的代码是*Host.CreateDefaultBuilder()*方法的简化版本。 它与2.x中的*WebHost.CreateDefaultBuilder()*方法类似，但是有一些有趣的更改，我将在稍后讨论。

```csharp
public static IHostBuilder CreateDefaultBuilder(string[] args)
{
    var builder = new HostBuilder();

    builder.UseContentRoot(Directory.GetCurrentDirectory());
    builder.ConfigureHostConfiguration(config =>
    {
        // 加载DOTNET_开头的环境变量及命令行参数
    });

    builder.ConfigureAppConfiguration((hostingContext, config) =>
    {
        // 读取Json文件，用户机密，环境变量及命令行参数
    })
    .ConfigureLogging((hostingContext, logging) =>
    {
        // 添加日志记录器，输出至控制台，调试记录器，事件源及事件记录（仅限于windows）
    })
    .UseDefaultServiceProvider((context, options) =>
    {
        // 配置依赖注入验证器（新功能）
    });

    return builder;
}
```

小结一下，[此方法与2.X差异](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.0#default-builder-settings)

-  读取`DOTNET_`前缀的环境变量来配置主机（Host）。
- 读取命令行来配置主机（Host）。
- 添加 `EventSourceLogger`  和 `EventLogLogger`  。
- ServiceProvider验证（可选）。
- 针对`Web Host`没有特定的配置（也就是说通用主机与Web主机配置上无差异）。

首先要关注的是如何设置主机配置。 在Web主机上，默认情况下配置使用带有`ASPNETCORE_`前缀的环境变量。 因此，设置`ASPNETCORE_ENVIRONMENT`环境变量将设置环境配置值。 对于通用主机，此前缀现在为`DOTNET_`，加上在运行时传递给应用程序的所有命令行参数。

> [**主机Host配置（比如从环境变量加载）**](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/environments?view=aspnetcore-2.2)应该与您的应用程序配置分开。

使用*ConfigureAppConfiguration()*来配置你的应用程序设置，从2.X开始没有变化，所以它仍然采用了*appsettings.json*文件，*appsettings.ENV.json*文件，[用户机密](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets)，环境变量和命令行参数。

通用主机的日志记录部分已在3.0中进行了扩展。 它仍然通过您的应用程序配置来配置[日志级别](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-2.2#log-filtering)的过滤，并添加控制台和调试记录器提供程序。 另外，它还添加了[EventSource](https://andrewlock.net/logging-using-diagnosticsource-in-asp-net-core/)日志记录提供程序，该提供程序用于与OS日志记录系统（例如Windows上的[ETW](https://docs.microsoft.com/en-us/windows/win32/etw/about-event-tracing)和Linux上的[LTTng](https://lttng.org/)）接口。 此外，仅在Windows上，记录器会添加[事件日志提供程序](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging/?view=aspnetcore-2.2#windows-eventlog-provider)，用于写入Windows事件日志。

最后，通用主机配置了依赖项注入的容器，并在开发环境运行时可以验证依赖注入，就像在2.x中一样。 目的是在获取的依赖注入的实例，在此实例中，您可以范围服务注入了Singleton服务。 在3.0中，通用主机还启用`ValidateOnBuild`，这是我将在后续文章中介绍的功能。

通用主机的另外一个关键点在于它是通用的，也就是说，它没有绑定与ASP.NET Core或HTTP相关的内容。 您可以将通用主机作为创建控制台应用程序，后台服务或典型ASP.NET Core应用程序的基础。 为了解决这个问题，在3.0中，还有一种附加方法可在顶部添加ASP.NET Core层-`ConfigureWebHostDefaults()`。

## 使用ConfigureWebHostDefaults恢复ASP.NET Core功能

这篇文章已经很长了，因此在这里我不会赘述。`ConfigureWebHostDefaults`扩展方法的作用是：在通用主机功能的基础上添加ASP.NET Core 层的功能。 简单地说，是将`Kestrel Web server`添加到通用主机，但是也包括[许多其他更改](https://github.com/aspnet/AspNetCore/blob/5b2f3fb5f7f24ac3e91c5150a55cc411b2b36b76/src/DefaultBuilder/src/WebHost.cs#L208)。 以下是该方法提供的说明（包括通过`GenericWebHostBuilder`提供的[功能](https://github.com/aspnet/AspNetCore/blob/f990751f54/src/Hosting/Hosting/src/GenericHost/GenericWebHostBuilder.cs)）：

- 将`ASPNETCORE_`前缀的环境变量添加到主机配置中（除了`DOTNET_`前缀的变量和命令行参数之外）。
- 添加[GenericWebHostService](https://github.com/aspnet/AspNetCore/blob/f990751f54/src/Hosting/Hosting/src/GenericHost/GenericWebHostedService.cs)。 这是`IHostedService`实现，它实际上运行ASP.NET Core服务器。 这是使ASP.NET Core可以重用通用主机的主要功能。
- 添加了一个额外的应用程序配置源，`StaticWebAssetsLoader`，用于处理Razor UI类库中的静态文件（css / js）。
- 使用默认的Kestrel配置（与2.x相同）。
- 添加  `HostFilteringStartupFilter`  （与2.x相同）。
- 当`ForwardedHeaders_Enabled`配置值为`true`，例如 `ASPNETCORE_FORWARDEDHEADERS_ENABLED`环境变量为`true`时，需要添加`ForwardedHeadersStartupFilter`。
- 当服务器是Windows时，开启`IIS`集成。
- 将[端点路由](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.0#differences-from-earlier-versions-of-routing)服务添加到DI容器。

上面方法步骤的大部分与ASP.NET Core 2.x中的相同，不同之处在于：用于将应用程序作为`IHostedService`运行的基础结构； 端点路由，此路由在3.0中全局启用（而不是仅在2.2中针对MVC /Razor Page启用）； 和`ForwardedHeadersStartupFilter`。

`ForwardedHeadersMiddleware`从1.0开始就存在，将应用程序托管在代理（例如 Nginx）之后面时是必需的，以确保您的应用程序可以处理 SSL-offloading并正确生成URL。 所更改的是，您只需设置环境变量即可将中间件配置为使用`X-Forwarded-For`和X-Forwarded-Proto标头。

## 总结

在本文中，我仅用两个文件深入研究了从ASP.NET Core 2.x到3.0的更改：*.csproj*项目文件和*Program.cs*文件。 从表面上看，对这些文件的更改很小，因此从2.x移植到3.0并不难。 这种简单性掩盖了内部的较大变化：**共享框架发生了重大变化，并且ASP.NET Core已在通用主机之上重建**。

我希望人们会遇到的最大问题是NuGet包之间的差异-一些应用程序将不得不删除对ASP.NET Core包的引用，同时添加对其他包的显式引用。 尽管不难解决，但对于不熟悉此更改的用户可能会造成混淆，因此应首先怀疑是否存在任何问题。