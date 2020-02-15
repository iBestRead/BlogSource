---
title: 使用Serilog记录MVC属性
tags: 
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 日志
  - Serilog

date: 2020-02-15 07:40:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Logging MVC properties with Serilog.AspNetCore](https://andrewlock.net/using-serilog-aspnetcore-in-asp-net-core-3-logging-mvc-propertis-with-serilog/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是[在ASP.NET Core 3.0中使用Serilog](/using-serilog-aspnetcore-in-asp-net-core-3/)第3篇:

1. [使用Serilog减少日志的详细程度](/using-serilog-aspnetcore-in-asp-net-core-3-reducing-log-verbosity/)
4. [使用Serilog记录路由的端点](/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/)
5. [使用Serilog记录MVC属性](/using-serilog-aspnetcore-in-asp-net-core-3-logging-mvc-propertis-with-serilog/)
6. [在Serilog日志中排除健康检查日志](/using-serilog-aspnetcore-in-asp-net-core-3-excluding-health-check-endpoints-from-serilog-request-logging/)

在本系列的[上一篇](/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/)中，我介绍了如何配置Serilog的`RequestLogging`中间件，来向Serilog的摘要日志中添加额外属性。 这些属性可以从`HttpContext`中获得，直接被中间件使用。

有一些属性（MVC特有的功能，例如*Action*名称，*RazorPages*处理程序名称或*ModelValidationState*等）仅在MVC上下文中可用，因此Serilog的中间件不能直接访问。

在本文中，我将介绍如何通过创建*Action*/*Page*的过滤器来记录这些属性，以方便中间件生成日志时使用。

> [Serilog的作者Nicholas Blumhardt在一篇文章中解决了这个问题](https://nblumhardt.com/2019/10/serilog-mvc-logging/)。 他在示例中创建了一个*attribute*，并在*actions*/*controllers*中使用。 本文并没有采用这种方式，我想实现一种更通用的解决方案。 另外还可以查看他[有关ASP.NET Core 3.0的Serilog相关文章](https://nblumhardt.com/2019/10/serilog-in-aspnetcore-3/)

<!-- more -->

## 记录MVC中的额外信息

ASP.NET Core中有很多功能被封装在MVC“基础架构”之内。 目前，**端点路由**这个功能已经被迁移到.NET Core框架中实现了。 ASP.NET Core团队一直在努力将更多的MVC特有功能（例如模型绑定或操作结果）从MVC内部移出，然后迁移到核心框架中。 有关更多信息，请参见[Ryan Nowak在NDC上对Houdini项目的讨论](https://www.youtube.com/watch?v=7dJBmV_psW0)。

就目前情况而言，MVC内部仍然有一些信息，是不太容易从外部获取的。 当使用Serilog的`RequestLogging`中间件时，以下信息就不太容易获取：

- HandlerName (`OnGet`) 
- ActionId (`1fbc88fa-42db-424f-b32b-c2d0994463f1`) 
- ActionName (`MyController.SomeApiMethod (MyTestApp)`) 
- RouteData (`{action = "SomeApiMethod", controller = "My", page = ""}`) 
- ValidationState (`True`/`False`) 

在[上一篇文章](/using-serilog-aspnetcore-in-asp-net-core-3-logging-the-selected-endpoint-name-with-serilog/)中，我介绍了如何使用`RequestLogging`中间件将额外附加信息写入Serilog的请求日志。 这仅适用于`HttpContext`中可访问的属性。 在这篇文章中，我将介绍如何在[action filter](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-3.0)中使用`IDiagnosticContext`将MVC特有的属性也添加到日志中。 还将介绍如何使用[page filter](https://docs.microsoft.com/en-us/aspnet/core/razor-pages/filter?view=aspnetcore-3.0#implement-razor-page-filters-globally)添加RazorPages特有的属性（例如HandlerName）。

## 使用Action过滤器记录MVC特有属性

过滤器相当于MVC框架中的一个微型的中间件管道。MVC中有多种类型的过滤器，每种类型的过滤器都在MVC过滤器管道中的不同点运行（有关更多详细信息，请参见此文章）。 在本文中，我们将使用最常见的过滤器之一，即动作过滤器。

