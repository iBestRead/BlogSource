---
title: ASP.NET Core in Action - 过滤器
tags: 
  - ASP.NET CORE
  - ASP.NET CORE IN ACTION
  - MVC

date: 2020-02-15 10:09:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [ASP.NET Core in Action - Filters]( https://andrewlock.net/asp-net-core-in-action-filters/ ) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

## 了解过滤器以及何时使用它们 

MVC过滤器管道的概念相对简单的，相当于在MVC的请求过程中，提供了很多“钩子”，如图1所示。当用户登录成功后，才允许新建或编辑商品，否则，重定向到登录页面。

如果没有过滤器，则需要在每个特定*Action*方法上添加很多相同的代码，以检查用户是否登录。即使采用了这种方式，当用户未登录时访问新建商品*Action*，`MvcMiddleware`中间件仍会执行模型绑定和验证的相关逻辑。

借助过滤器，可以使用MVC请求中那些的“钩子”，让所有的请求都可以运行共用的代码。 例如下面这些场景：

- 确保用户已登录后，再执行对应的*Action*方法、模型绑定、模型验证
- 自定义*Action*的输出格式
- 在*Action*执行前，处理模型验证失败的情况
- 在*Action*中捕获异常，并以特殊方式处理它们

<!-- more -->

![图1](https://cdn.ibestread.com/img/ASP.NET-Core-MVC---Filters.png)

图1中，在处理请求的过程中，过滤器会在多个不同的位置被调用。

MVC筛选器管道类似于中间件管道，但仅限于`MvcMiddleware`。 像中间件一样，过滤器非常适合处理程序中的切片问题，并且有助于减少重复的代码工作。

## MVC过滤器管道

如图1所示，MVC筛选器在请求过程中，在多个不同位置被调用。图2显示了五种不同类型的过滤器，每种过滤器会在`MvcMiddleware`的不同阶段运行。

针对不同的阶段的应用场景，例如模型绑定，*Action*的执行，*result*的执行等，应该使用不同的过滤器：

-  ***Authorization filters***：它是在管道中运行的第一个过滤器，用于保护*Action*。 如果请求未经授权，该过滤器会短路该请求，从而阻止剩余过滤器运行。
- ***Resource filters*** ：它在*Authorization*过滤器之后运行，还可以在管道的末尾运行，就像中间件可以处理传入的*request*和传出*response*一样。 另外，它也可以短路请求，并直接返回*response*。 由于它在管道较前的位置，*Resource*过滤器可以有多种用途。 可以用来判断*Action*是否支持传入的*Content-type*，如果不支持，则阻止*Action*执行；又或者在模型绑定之前针对不同的*Action*实现不同的模型绑定方式。
- ***Action filters***：*Action*过滤器会在*Action*执行之前和之后运行。 由于模型已绑定完成，因此*Action*过滤器可以在*Action*执行前对*Action*方法的参数进行操作，或者使*Action*短路并直接返回*IActionResult*。
- ***Exception filters***：*Exception*过滤器可以捕获管道中发生的异常，并对其进行适当的处理。可以编写自定义的异常处理代码。 例如，针对*Web API Action*和*MVC Action*的中产生异常，分别使用不同异常格式化方式。
- ***Result filters***：*Result*过滤器在*Action*的*IActionResult*执行之前和执行之后运行。 可以控制结果的执行，或者短路。

![图2](https://cdn.ibestread.com/img/ASP.NET-Core-MVC-Five-Filters.png)

> **注意**：图2中 MVC过滤器管道包括五个不同的阶段。 其中3个过滤器，“***Resource filters***”、“***Action filters***”、“***Result filters***”会在其它管道之前和之后各运行一次，共两次。

具体使用哪种过滤器，取决于您的需求。 如果希望尽早地短路请求， *Resource filters*适用； 如果希望访问*Action*方法的参数，*Action filters*适用。