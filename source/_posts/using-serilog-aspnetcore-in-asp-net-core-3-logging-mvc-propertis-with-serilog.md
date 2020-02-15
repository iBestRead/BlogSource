---
title: 使用Serilog记录MVC属性
tags: 
  - ASP.NET CORE
  - ASP.NET CORE 3.0
  - 日志
  - Serilog

date: 2020-02-15 14:33:00
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

过滤器相当于MVC框架中的一个微型的中间件管道。MVC中有多种类型的过滤器，它们会在过滤器管道的不同位置运行（有关MVC过滤器更多详细信息，[请参见此文章](/asp-net-core-in-action-filters/)）。 在本文中，我们将使用最常见的过滤器之一，即***Action filter***。

*Action*过滤器会在MVC*Action*方法执行前和执行后运行。 这个过滤器可以访问到许多MVC特有的属性，例如将要执行的*Action*及执行它的参数。

下面的*Action*过滤器实现了`IActionFilter`接口。 当*Action*的方法被执行之前，会先调用此过滤器的`OnActionExecuting`方法，在此方法中获取MVC特有的属性，添加到`IDiagnosticContext`中。

```csharp
public class SerilogLoggingActionFilter : IActionFilter
{
    private readonly IDiagnosticContext _diagnosticContext;
    public SerilogLoggingActionFilter(IDiagnosticContext diagnosticContext)
    {
        _diagnosticContext = diagnosticContext;
    }

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _diagnosticContext.Set("RouteData", context.ActionDescriptor.RouteValues);
        _diagnosticContext.Set("ActionName", context.ActionDescriptor.DisplayName);
        _diagnosticContext.Set("ActionId", context.ActionDescriptor.Id);
        _diagnosticContext.Set("ValidationState", context.ModelState.IsValid);
    }

    // Required by the interface
    public void OnActionExecuted(ActionExecutedContext context){}
}
```

在`Startup.ConfigureServices()`中注入MVC服务时，同时注入全局范围过滤器：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers(opts =>
    {
        opts.Filters.Add<SerilogLoggingPageFilter>();
    });
    // ... other service registration
}
```

> 无论您使用的是`AddControllers`，`AddControllersWithViews`，`AddMvc`还是`AddMvcCore`，都可以以相同的方式注入全局过滤器。

完成此配置后，再次访问MVC的*Controller*，则会在Serilog请求日志消息中看到额外信息（`ActionName`，`ActionId`和`RouteData`，`ValidationState`）：

![](https://cdn.ibestread.com/img/serilog_action_name.png)

您可以使用此方式，添加您所需的任何其他信息添加到日志中。 注意：**不要记录敏感信息或个人身份信息**！

> [Nicholas Blumhardt的文章](https://nblumhardt.com/2019/10/serilog-mvc-logging/)中建议，*Action*过滤器从`ActionFilterAttribute`派生的，方便其用作于*controller*和*action*。这样做意味着，必须使用`IServiceProvider`来获取单例的*IDiagnosticContext*对象。
>
> 上面的方法改为使用构造函数注入，因此不能当*Attribute*使用。 另外，生命周期是*scoped*，非*singleton*，因此每次请求都会创建一个新实例。

如果要记录MVC过滤器管道中其他位置的相关属性，可以采用类似的方式实现其它过滤器，例如*Resource filter*、*Result filter*、*Authorization filter*。

## 使用Page过滤器记录RazorPages属性

上面的`IActionFilter`只能在*MVC controller*和*API controller*上运行，但不能在`RazorPages`上运行。 如果要记录为指定Razor页面对应的`HandlerName`，则需要创建一个自定义`IPageFilter`。

*Page filters*类似于*Action filters*，但只仅适用于Razor Pages 。 下面的示例从`PageHandlerSelectedContext`获取`HandlerName`。最后调用`IDiagnosticContext .Set()`来记录属性。

```csharp
public class SerilogLoggingPageFilter : IPageFilter
{
    private readonly IDiagnosticContext _diagnosticContext;
    public SerilogLoggingPageFilter(IDiagnosticContext diagnosticContext)
    {
        _diagnosticContext = diagnosticContext;
    }

    public void OnPageHandlerSelected(PageHandlerSelectedContext context)
    {
        var name = context.HandlerMethod?.Name ?? context.HandlerMethod?.MethodInfo.Name;
        if (name != null)
        {
            _diagnosticContext.Set("RazorPageHandler", name);
        }
    }

    // Required by the interface
    public void OnPageHandlerExecuted(PageHandlerExecutedContext context){}
    public void OnPageHandlerExecuting(PageHandlerExecutingContext context) {}
}
```

请注意，之前编写的`IActionFilter`不会在Razor Pages上运行，因此，如果也想为RazorPages记录诸如`RouteData`或`ValidationState`之类的其他详细信息，那么也需要在此处添加那些代码。 `context`属性包含大多数需要的属性，例如`ModelState`和`ActionDescriptor`。

接下来，您需要在`Startup.ConfigureServices()`方法中注册*Page filters*：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddMvcCore(opts =>
    {
        opts.Filters.Add<SerilogLoggingPageFilter>();
    });
    services.AddRazorPages();
}
```

添加过滤器后，对Razor Pages的请求额外属性将会添加到Serilog请求日志中。 请参见下图中的`RazorPageHandler`属性：

![](https://cdn.ibestread.com/img/serilog_razor_handler.png)

## 总结

默认情况下，当用Serilog的请求日志记录中间件替换ASP.NET Core基础日志组件时，会丢失一些信息（与开发环境的默认配置相比）。 在本文中，我介绍了如何自定义Serilog的`RequestLoggingOptions`来添加MVC特有的其他属性。

需要添加MVC相关的属性到Serilog请求日志中，创建`IActionFilter`并使用`IDiagnosticContext.Set()`添加属性。

需要添加Razor页面相关的属性添加到Serilog请求日志中，创建`IPageFilter`并使用`IDiagnosticContext`添加属性。