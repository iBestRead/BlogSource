---
title: .Net Core 3.0的新特性-本地工具
tags: 
  - .NET CORE
  - .NET CORE 3.0
  - 工具

date: 2020-02-03 12:45:00
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

# 全局工具， 

.NET Core 2.1引入了[全局工具](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools)的概念，它们是靠`.NET Core SDK`来安装的CLI工具（实际上是控制台应用程序）。 这些工具在您的计算机上是全局可用的。

最近，我通过使用tools-path选项将全局工具安装到项目文件夹中，然后从那里运行工具，来将Cake全局工具用于新版本。 通过这种方式安装到项目文件夹中意味着您可以为每个项目使用不同版本的全局工具，而不必被迫一次更新所有工具。