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

date: 2020-02-05 11:32:00
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
4. [对比IHostingEnvironment与IHostEnvironment .NET及Core 3.0中的过时类型](/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/)
5. [不要在Startup类的构造函数中使用依赖注入](/avoiding-startup-service-injection-in-asp-net-core-3/)
6. [将末端中间件转换为端点路由](/converting-a-terminal-middleware-to-endpoint-routing-in-aspnetcore-3/)
7. [将集成测试升级至.NET Core 3.0](/converting-integration-tests-to-net-core-3/)

