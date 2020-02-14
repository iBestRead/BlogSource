---
title: 系列-在ASP.NET Core 3.0中使用Serilog
tags: 
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 日志
  - 系列

categories:
  - 系列
  - 在ASP.NET Core 3.0中使用Serilog.AspNetCore

date: 2020-02-13 07:30:00
---

> 译者:  [Akini Xu](/)
>
> 原文:  [Series: Using Serilog.AspNetCore in ASP.NET Core 3.0](https://andrewlock.net/series/using-serilog-aspnetcore-in-asp-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

ASP.NET Core框架中是包含日志记录基础组件的。 您可以方便地通过使用框架的日志组件，获得所有日志。 只是有时候日志数量过多。

在这个系列中，我将介绍如何[使用Serilog记录ASP.NET Core的请求日志](https://github.com/serilog/serilog-aspnetcore#request-logging)。 在第一篇文章中，会介绍如何将Serilog `RequestLoggingMiddleware`中间件添加到您的应用程序管道中，以及带来的好处。 在后续文章中，我将进一步介绍如何定制中间件来向日志中添加其他额外信息。

> Serilog的作者Nicholas Blumhardt，写了一篇博客“[如何在ASP.NET Core 3.0中使用Serilog](https://nblumhardt.com/2019/10/serilog-in-aspnetcore-3)”。强烈建议您阅读。 在他的文章中找到我在本系列文章中谈论的大部分内容。

本系列文章列表:

1. [使用Serilog减少日志的详细程度](/using-serilog-aspnetcore-in-asp-net-core-3-reducing-log-verbosity/)
4. [使用Serilog记录路由的端点](/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/)
5. [使用Serilog记录MVC属性](/using-serilog-aspnetcore-in-asp-net-core-3-logging-mvc-propertis-with-serilog/)
6. [在Serilog日志中排除健康检查日志](/using-serilog-aspnetcore-in-asp-net-core-3-excluding-health-check-endpoints-from-serilog-request-logging/)

