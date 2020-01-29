---
title: 系列:探索 ASP.NET Core 3.0
top_img: https://cdn.ibestread.com/img/banner_aspnetcore3.jpg
tags: 
  - dotnet-core
  - asp-dotnet-core
  - asp-dotnet-core-3

categories:
  - 系列
  - 探索ASP.NETCore3.0
date: 2020-01-31 10:31:00
---

> 原文:  [Exploring ASP.NET Core 3.0](https://andrewlock.net/series/exploring-asp-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>
> 译者:  [Akini Xu](https://blog.ibestread.com)

在这个系列中，我们来探讨一些ASP.NET Core 3.0带来的变化及新特性。但是不会关注那些较为宽泛的特性，因为有已经有很多文章对其进行介绍，比如  [server-side Blazor](https://docs.microsoft.com/en-us/aspnet/core/blazor/?view=aspnetcore-3.0) 或 [gRPC client factory](https://docs.microsoft.com/en-us/aspnet/core/tutorials/grpc/grpc-start)  等。 

相反地，我们会关注一些非重要的特性和一些小的变化。你可能会忽视这些小的变化，如果你不去了解它们，就可能掉进坑里。最后还会介绍一些非常有意思的新特性。

本系列文章列表:

1. [探索新的.csproj文件,Program.cs及通用主机](https://blog.ibestread.com/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)
2. [ASP.Net  Core 3.0的`Startup.cs`在不同项目类型中的差异](https://blog.ibestread.com/comparing-startup-between-the-asp-net-core-3-templates/)
3. [ASP.Net  Core 3.0的新特性-Service provider validation](https://blog.ibestread.com/new-in-asp-net-core-3-service-provider-validation)
4. [ASP.Net  Core 3.0中应用程序启动时运行异步任务](https://blog.ibestread.com/running-async-tasks-on-app-startup-in-asp-net-core-3)
5. [介绍`IHostLifetime`及与通用主机间的作用关系](https://blog.ibestread.com/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions)
6. [ASP.Net  Core 3.0的新特性-启动时的结构化日志](https://blog.ibestread.com/new-in-aspnetcore-3-structured-logging-for-startup-messages)
7. [ASP.Net  Core 3.0的新特性-本地工具](https://blog.ibestread.com/new-in-net-core-3-local-tools)

