---
title: .Net Core 3.0的新特性-本地工具
tags: 
  - .NET CORE
  - .NET CORE 3.0
  - 工具
  - 全局工具

date: 2020-02-04 11:15:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Exploring the new project file, Program.cs, and the generic host](https://andrewlock.net/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [探索 ASP.NET Core 3.0](https://blog.ibestread.com/exploring-asp-net-core-3) 第7篇:

1. [`ASP.Net Core 3.0`.csproj文件,Program.cs及通用主机](/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)
2. [`ASP.Net Core 3.0`Startup.cs在不同类型项目中的差异](/comparing-startup-between-the-asp-net-core-3-templates/)
3. [`ASP.Net Core 3.0`新特性-Service provider validation](/new-in-asp-net-core-3-service-provider-validation/)
4. [`ASP.Net Core 3.0`应用程序启动时运行异步任务](/running-async-tasks-on-app-startup-in-asp-net-core-3/)
5. [介绍IHostLifetime及与通用主机间的作用关系](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/)
6. [`ASP.Net Core 3.0`新特性-启动时的结构化日志](/new-in-aspnetcore-3-structured-logging-for-startup-messages/)
7. [`.Net Core 3.0`新特性-本地工具](/new-in-net-core-3-local-tools)

在本文中，我将探讨.NET Core 3.0中引入的新特性: 本地工具。 我将展示如何使用`dotnet-tools`安装和运行本地工具，描述如何使用多个清单，并描述如何安装这些工具。

<!-- more --> 

# 全局工具， 本地tools文件夹工具，本地工具

.NET Core 2.1引入了[全局工具](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools)的概念，它们是靠`.NET Core SDK`来安装的CLI工具（实际上是控制台应用程序）。 这些工具在您的计算机上是全局可用的。

通过使用`tools-path`选项将全局工具安装到项目文件夹中，然后从那里[运行Cake全局工具来构建系统](/how-to-build-with-cake-on-linux-using-cake-coreclr-or-the-cake-global-tool/)。 通过这种方式安装到项目文件夹中，可以使每个项目使用不同版本的全局工具，而不需要一次更新所有工具。

但是，手动指定文件夹安装全局工具这种方式总感觉有些麻烦。在.NET Core 3.0中，已经支持了“特定项目”的全局工具这种新特性。这里[有篇文章介绍这个新特性](https://stu.dev/dotnet-core-3-local-tools/)

## .NET Core 3.0中的本地工具

在.NET Core 3.0中，您现在可以通过创建dotnet-tools清单方式，来指定特定项目所需的全局工具。 它一个JSON文件，位于您的代码仓库中，最好受源代码代码管理。 您可以在代码仓库的根目录中运行以下命令来创建新的工具清单：

```bash
dotnet new tool-manifest
```

默认情况下，这会在代码仓库的根目录的*.config*文件夹内创建以下清单JSON文件*dotnet-tools.json*：

```json
{
  "version": 1,
  "isRoot": true,
  "tools": { }
}
```

默认清单不包含任何工具，但是您可以通过运行`dotnet tool`安装来安装新工具（不需要像.NET Core 2.x中`-g`或 `--tool-path`那样的参数）。 因此，您可以通过运行以下命令为项目安装Cake全局工具：

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "cake.tool": {
      "version": "0.35.0",
      "commands": [
        "dotnet-cake"
      ]
    }
  }
}
```

当其他人隆仓库并想要运行Cake工具时，他们可以运行以下命令来首先还原该工具，然后运行它：

```bash
# Restore the tool NuGet packages
dotnet tool restore
# Execute the tool associated with command "dotnet-cake" and pass the arguments: --version
dotnet tool run dotnet-cake --version
# or you can use:
dotnet dotnet-cake --version
# or even shorter:
dotnet cake --version
```

对于像Cake这样的构建工具，您可能需要或需要为不同的项目安装不同的版本，.NET Core 3本地工具非常有用。 我在之前的文章[在项目的tools文件夹安装全局工具](https://blog.ibestread.com/how-to-build-with-cake-on-linux-using-cake-coreclr-or-the-cake-global-tool/#使用Cake全局工具在Linux上构建)的方式是可行的，但是作一些额外操作才能确保安装正确的版本。 NET Core 3会让一切变得更简单，我将在以后的文章中展示。

## .NET Core 工具如何运行的

在本节中，我再使用本地工具后，探讨几个问题。 本地工具安装在哪里？清单文件中的其他属性是什么？我可以将清单文件放在其他地方吗？

### dotnet-tools.json 清单

当您使用`dotnet new tool-manifest`创建新清单时，将获得如下所示的JSON文件：

```json
{
  "version": 1,
  "isRoot": true,
  "tools": { }
}
```

`version`属性指的是`dotnet-tools`的版本，是架构版本，而不是文件版本，因此您需要将其设置为`1`。以后更高版本的.NET Core SDK可能会更新`dotnet-tools`架构以添加其它功能，此处`version`属性就是来确定使用哪个架构版本的`dotnet-tools`。

`isRoot`属性与`dotnet tool`命令搜索清单的方式有关。 稍后，我将详细介绍它，但是总而言之，`isRoot`的意思是“停止搜索其他文件夹，这里就是全部配置”。 它是“根”清单，即顶层清单。

### .NET Core Sdk如何查找清单文件

当需要查找*dotnet-tools.json*清单文件时，`dotnet`工具命令会在多个位置中进行检索：

1. 当前目录中的*.config*文件夹(`./.config/dotnet-tools.json`)
2. 当前目录（`./dotnet-tools.json`）
3. 父级文件夹（`../dotnet-tools.json`）
4. 再向上的父级只到根目录

一旦找到`isRoot`为`true`的*dotnet-tools.json*清单，它将立即停止搜索。所有的可用工具 **=** 根清单中列出的工具 **+** 在搜索过程中找到的清单中列出的所有工具。

您可以使用`dotnet tool list`命令，查看在指定文件夹中可用的本地工具。 

举个例子：假设在*.config*文件夹中有一个非根清单，该清单需要Cake全局工具。 在当前目录中也有一个非根清单，该清单安装`dotnetsay`全局工具，在父级目录中又有一个根清单，该清单安装`dotnet-tinify`工具。 运行`dotnet tool list`命令，会显示这三个工具均可用：

```bash
Package Id         Version      Commands           Manifest
-----------------------------------------------------------------------------------------------------
cake.tool          0.35.0       dotnet-cake        C:\repos\test\.config\dotnet-tools.json
dotnetsay          2.1.4        dotnetsay          C:\repos\test\dotnet-tools.json
dotnet-tinify      0.2.0        dotnet-tinify      C:\repos\dotnet-tools.json
```

优先级非常很重要，它知道什么时候停止搜索。同时它也可以处理，多个清单中定义的工具有不同版本的情况。 在这种情况下，以第一个清单中的版本为准。 因此，如果*.config*文件夹清单中包含`dotnetsay`工具的`1.0.0`版本，则`dotnet tool list`列表将输出以下内容：

```json
Package Id         Version      Commands           Manifest
-----------------------------------------------------------------------------------------------------
cake.tool          0.35.0       dotnet-cake        C:\repos\test\.config\dotnet-tools.json
dotnetsay          1.0.0        dotnetsay          C:\repos\test\.config\dotnet-tools.json
dotnet-tinify      0.2.0        dotnet-tinify      C:\repos\dotnet-tools.json
```

> 注意，上面` Version `和`Manifest`列信息。

**要点**：本地工具的清单在本地，已通过源代码控制签入，并且不依赖其他内容（例如，父文件夹中的清单）。 **强烈建议**仅使用一个清单，确保`isRoot`为`true`，并将其放置在*.config*文件夹或项目的根目录中。

### .NET Core 本地工具安装在哪

[答案](https://github.com/dotnet/cli/issues/10288#issuecomment-483449922)是它们安装在[全局NuGet软件包文件夹](https://docs.microsoft.com/en-us/nuget/consume-packages/managing-the-global-packages-and-cache-folders)中。 .NET Core全局/本地工具只是作为特殊NuGet软件包分发的控制台应用程序。 因此，它们像正常的NuGet包一样被下载到全局文件夹并解压缩。 如果在多个清单中指定了同一版本的工具，则仅在全局Nuget软件包文件夹中安装一次。

当您运行全局工具时，它将通过NuGet程序包运行应用程序。 例如，如果您安装dotnet-serv全局工具：

```bash
dotnet tool install dotnet-serve
```

并运行：

```bash
dotnet tool run dotnet-serve
# or you can use:
dotnet dotnet-serve
# or even shorter:
dotnet serve
```

然后在*进程资源管理器*中查看，您可以看到该工具是从*~/ .nuget / packages*文件夹运行的，用于该工具的已安装版本（在本例中为1.4.1）

![](https://cdn.ibestread.com/img/dotnet_local_tool.png)

指定的工具从共享的NuGet缓存中运行，使用 `dotnet tool uninstall <toolname>`命令，可以从清单中卸载工具。但只是从*dotnet-tools.json*清单中删除该条目，NuGet包仍处于缓存状态， 如果想从系统中完全删除该工具，则需要[清除NuGet缓存](https://docs.microsoft.com/en-us/nuget/consume-packages/managing-the-global-packages-and-cache-folders)。

## 总结

在这篇文章中，描述了.NET Core 3.0中引入的新的本地工具功能。 此功能使您可以在项目中包含清单，该清单列出了所需的.NET Core CLI工具。 可以为不同的项目使用不同的工具（或不同版本的工具）。 我展示了如何安装和运行本地工具，解释了清单文件的格式，以及如何使用多个清单文件（**不推荐**）。 最后，我通过运行全局NuGet包缓存中的工具描述了本地工具的工作方式。