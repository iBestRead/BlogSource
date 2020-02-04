---
title: 系列-升级至 ASP.NET Core 3.0
tags: 
  - .NET CORE
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 系列

categories:
  - 系列
  - 升级至 ASP.NET Core 3.0

date: 2020-02-04 11:30:00
---

> 译者:  [Akini Xu](/)
>
> 原文:  [Series: Upgrading to ASP.NET Core 3.0](https://andrewlock.net/series/upgrading-to-asp-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

在这个系列中，我讨论将ASP.NET Core 2.x应用程序和库升级到ASP.NET Core 3.0时要注意的一些事项。本系列不会讨论，比如  [server-side Blazor](https://docs.microsoft.com/en-us/aspnet/core/blazor/?view=aspnetcore-3.0) 或 [gRPC client factory](https://docs.microsoft.com/en-us/aspnet/core/tutorials/grpc/grpc-start)  等新特性的使用。 

相反地，我会关注于一些基础知识，例如调整目标框架，转换新的端点路由系统，移除过时的类型和方法，以及在升级过程中遇到的问题。

本系列文章列表:

1. [转换.NET Standard 2.0类库到.NET Core 3.0](/converting-a-netstandard-2-library-to-netcore-3/)
4. [对比IHostingEnvironment与IHostEnvironment .NET Core 3.0中的过时类型](/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/)
5. [`ASP.Net Core 3.0`新特性-Service provider validation](/new-in-asp-net-core-3-service-provider-validation/)
6. [`ASP.Net Core 3.0`应用程序启动时运行异步任务](/running-async-tasks-on-app-startup-in-asp-net-core-3/)
7. [介绍IHostLifetime及与通用主机间的作用关系](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/)
8. [`ASP.Net Core 3.0`新特性-启动时的结构化日志](/new-in-aspnetcore-3-structured-logging-for-startup-messages/)
9. [`.Net Core 3.0`新特性-本地工具](/new-in-net-core-3-local-tools)

