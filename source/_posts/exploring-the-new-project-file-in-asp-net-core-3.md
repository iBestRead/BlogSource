---
title: ASP.Net Core 3.0中的.csproj文件
tags: 
  - dotnet core
  - asp dotnet core
  - asp dotnet core 3.0
  - csproj

categories:
  - 系列
  - 探索ASP.NETCore3.0

date: 2020-01-31 10:31:58
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Exploring the new project file, Program.cs, and the generic host](https://andrewlock.net/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [探索 ASP.NET Core 3.0](https://blog.ibestread.com/exploring-asp-net-core-3) 第1篇:

1. ASP.Net Core 3.0中的.csproj文件(本文)
2. [ASP.Net Core 3.0中的Program.cs文件](https://blog.ibestread.com/exploring-the-program-file-in-asp-net-core-3/)
3. [ASP.Net Core 3.0中的通用主机](https://blog.ibestread.com/exploring-the-generic-host-in-asp-net-core-3/)
4. [ASP.Net Core 3.0的Startup.cs在不同项目类型中的差异](https://blog.ibestread.com/comparing-startup-between-the-asp-net-core-3-templates/)
5. [ASP.Net Core 3.0的新特性-Service provider validation](https://blog.ibestread.com/new-in-asp-net-core-3-service-provider-validation)
6. [ASP.Net Core 3.0中应用程序启动时运行异步任务](https://blog.ibestread.com/running-async-tasks-on-app-startup-in-asp-net-core-3)
7. [介绍IHostLifetime及与通用主机间的作用关系](https://blog.ibestread.com/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions)
8. [ASP.Net Core 3.0的新特性-启动时的结构化日志](https://blog.ibestread.com/new-in-aspnetcore-3-structured-logging-for-startup-messages)
9. [ASP.Net Core 3.0的新特性-本地工具](https://blog.ibestread.com/new-in-net-core-3-local-tools)

在这篇文章中我们来看看`ASP.NET Core 3.0`的应用程序的基础组件 : `.csproj`的项目文件和`Program.cs`启动文件。我将介绍`ASP.NET Core 3.0`与`ASP.NET Core 2.X`的差异，并讨论相关APIs的使用变化。 

 <!-- more --> 

## 介绍

`.NET Core 3.0`的核心是预计于9月23日的 [.NET Conf](https://www.dotnetconf.net/) 期间发布，现在预览版（`Preview 8`）现已发布。最终` Release版 `与`Preview版`不会有太多的差异，我们现在来看看到底带来了些什么。 

`.NET Core 3.0`的主要功能集中在Windows桌面应用程序可以在`.NET Core`上运行，另外`ASP.NET Core`也有很多新功能。其中最大的新功能是 [server-side Blazor](https://docs.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-3.0#server-side) （~~我更感兴趣的客户端版本个人，这是不完全产品尚未推出~~目前已经更新）。 

在这篇文章中,我们着重关注下面2个基础设施的变化:

- Nuget中的 [Microsoft.AspNetCore.App metapackage](https://andrewlock.net/the-microsoft-aspnetcore-all-metapackage-is-huge-and-thats-awesome-thanks-to-the-net-core-runtime-store-2/) 不再可用.
- `ASP.NET Core` 使用通用主机( [Generic Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/generic-host?view=aspnetcore-3.0) )替代了Web主机([Web Host](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/web-host?view=aspnetcore-3.0) )。

>  如果你打算从ASP.NET Core 2.x应用程序升级到3.0，则需要仔细查看微软的官方迁移指导( [the migration guidance on docs.microsoft.com](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30) )。 

在这篇文章中，我们创建一个`ASP.NET Core 3.0`的应用程序（例如：使用` dotnet new webapi `），并来看看`.csproj`文件，`Program.cs`中的变化。**在后面的文章中，我们会使用不同的项目模板（`Web`，`webapi`，`mvc`等）创建项目，并比较`Startup.cs`文件的变化差异。** 

## 新的项目文件及目标框架

创建一个新的ASP.NET Core项目文件后，查看`.csproj`文件，代码如下：

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.0</TargetFramework>
  </PropertyGroup>

</Project>
```

与`ASP.NET Core 2.X`对比后，发现有如下的相同点及差异：

- `<Project>`中的`Sdk`节点依然是` Microsoft.Net.Sdk.Web `。在新旧的项目文件中2者是一样的。
- `<TargetFramework>` 由` netcoreapp2.1 `或` netcoreapp2.2 `改为` netcoreapp3.0 `。原因是新版本依赖了`.Net Core 3.0`。
-  无需再对`Microsoft.AspNetCore.App`元包进行引用。

最有趣变化是最后一点。在ASP.NET Core 2.1/ 2.2，你可以引用*shared framework*的元数据包*Microsoft.AspNetCore.App*，正如我在以前的文章[shared framework](https://andrewlock.net/exploring-the-microsoft-aspnetcore-app-shared-framework-in-asp-net-core-2-1-preview-1/)中描述。*shared framework*提供了许多好处，例如，避免手动安装所有单独的软件包，并允许运行时的向前更新。 

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
