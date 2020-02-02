---
title: ASP.Net Core 3.0的Startup.cs在不同项目类型中的差异
tags: 
  - dotnet-core
  - asp-dotnet-core
  - asp-dotnet-core-3
  - Startup

date: 2020-01-31 10:31:55
---

> 译者:  [Akini Xu](/)
>
> 原文:  [Comparing Startup.cs between the ASP.NET Core 3.0 templates](https://andrewlock.net/comparing-startup-between-the-asp-net-core-3-templates/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [探索 ASP.NET Core 3.0](/exploring-asp-net-core-3) 第2篇:

1. [ASP.Net Core 3.0中的.csproj文件,Program.cs及通用主机](/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)
2. ASP.Net Core 3.0的Startup.cs在不同项目类型中的差异(本文)
3. [ASP.Net Core 3.0的新特性-Service provider validation](/new-in-asp-net-core-3-service-provider-validation/)
4. [ASP.Net Core 3.0中应用程序启动时运行异步任务](/running-async-tasks-on-app-startup-in-asp-net-core-3/)
5. [介绍IHostLifetime及与通用主机间的作用关系](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/)
6. [ASP.Net Core 3.0的新特性-启动时的结构化日志](/new-in-aspnetcore-3-structured-logging-for-startup-messages/)
7. [ASP.Net Core 3.0的新特性-本地工具](/new-in-net-core-3-local-tools)

.NET Core 3.0 SDK比以前的版本提供了更多的项目模板。 在本文中，我将比较使用不同的项目模板生成ASP.NET Core 3应用程序，并来看看其中一些新的服务、中间件的配置方法。

首先，我们来看看有哪些项目模板：

-  [ASP.NET Core Empty template](https://andrewlock.net/comparing-startup-between-the-asp-net-core-3-templates/#the-asp-net-core-empty-template) 
-  [ASP.NET Core Web API template](https://andrewlock.net/comparing-startup-between-the-asp-net-core-3-templates/#the-asp-net-core-web-api-template) 
-  [ASP.NET Core Web App (Model-View-Controller) template](https://andrewlock.net/comparing-startup-between-the-asp-net-core-3-templates/#the-asp-net-core-web-app-mvc-template) 
-  [ASP.NET Core Web App (Razor) template](https://andrewlock.net/comparing-startup-between-the-asp-net-core-3-templates/#asp-net-core-web-app-razor-template) 
- ......

这里没有列举所有的模板，你可以使用` dotnet new list `命令来查看所有模板。

<!-- more --> 

## ASP.NET Core Empty template

您可以通过运行` dotnet new web `来创建一个空模板项目，该模板非常简洁。*Program.cs*中使用了标准的通用主机，另外 *Startup.cs* 代码如下：

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services) { }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseRouting();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapGet("/", async context =>
            {
                await context.Response.WriteAsync("Hello World!");
            });
        });
    }
}
```

与ASP.NET Core 2.x相比，主要区别在于对[端点路由](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-3.0)的显示调用。 它是在2.2中引入的，但是只能用于MVC控制器。 在3.0中，端点路由是首选方法，上面代码演示了最基本的设置。

**端点路由** 将配置使用哪个“端点”的与该端点实际执行的逻辑分离。 端点是由路径及调用时要执行的逻辑组成。 可以是控制器上的MVC操作，也可以是简单的lambda，如上面代码所示，使用`MapGet()`方法创建一个路径为`/`的端点。

扩展方法`UseRouting()`的作用是确认入站的请求，并决定由哪个端点响应该请求。所有位于`UseRouting()`方法后的中间件，最终都会在端点路由指定的逻辑中运行。`UseEndpoints()`负责配置端点，并执行。

如果你是第一次接触端点路由，我建议你先看看这2篇博客， [this post by Areg Sarkissian](https://aregcode.com/blog/2019/dotnetcore-understanding-aspnet-endpoint-routing/) ， [this post by Jürgen Gutsch](https://asp.net-hacker.rocks/2019/04/10/routed-middlewares.html) 。这2篇博文后面大家有兴趣我再来翻译。

## ASP.NET Core Web API template

使用` dotnet new webapi `创建的Web API模板项目是最复杂的。它包含了一个简单的[带有HttpGet的Controller](https://github.com/aspnet/AspNetCore/blob/v3.0.0-preview9.19424.4/src/ProjectTemplates/Web.ProjectTemplates/content/WebApi-CSharp/Controllers/WeatherForecastController.cs)。另外 *Startup.cs*文件也比空模板项目稍微复杂点，但大体上是相同的。代码如下：

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseHttpsRedirection();

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

上面代码中，可以看到`IConfiguration`注入到了的构造函数中，但未曾使用它。 不过在实际的应用程序中，你肯定需要访问它，因为它才能读取配置。

在`ConfigureServices`方法中，调用`AddControllers()`这个扩展方法的，这是ASP.NET Core 3.0中的新增功能。 在2.x中，通常会在ASP.NET Core应用程序中调用`services.AddMvc()`。 但是，这个方法是在使用MVC时所使用的，例如Razor Pages和View渲染。 如果只是仅仅创建Web API，那么`services.AddMvc()`中的一些服务则是多余的。

为了解决这个问题，我在[之前的文章](https://andrewlock.net/removing-the-mvc-razor-dependencies-from-the-web-api-template-in-asp-net-core/)中演示了如何创建一个精简版的`AddMvc()`，仅添加了创建Web API所需要的内容。 现在，`AddControllers()`扩展方法就做到这一点-它仅添加了使用Web API控制器所需的服务，仅此而已。 因此，例如，您获得了Authorization，Validation，formatters和CORS，但与Razor Pages或视图呈现无关。 有关所包含内容的完整详细信息，[请参见GitHub上的源代码](https://github.com/aspnet/AspNetCore/blob/0303c9e90b5b48b309a78c2ec9911db1812e6bf3/src/Mvc/Mvc/src/MvcServiceCollectionExtensions.cs#L87)。

与空模板相比，Web API模板还添加了几个中间件。

在开发环境中运行时，开启开发人员异常查看页面。注意此页面只在开发环境生效，其他环境没有此页面。 `ApiController`会将错误转换为标准格式的[异常详细信息](https://docs.microsoft.com/en-us/aspnet/core/web-api/?view=aspnetcore-3.0#problem-details-for-error-status-codes)。

接下来是HTTPS重定向中间件，它确保请求是在安全域名上发出。（最佳实践）。 

然后，就是路由中间件，以便后续的中间件确定哪个端点来运行逻辑。

Authorization中间件是3.0中的新增功能，并且在很大程度上归功于端点路由的引入。 您仍然可以使用`[Authorize]`特性（ **attributes 后面会统称为特性**），3.0中这些特性的最终的运行逻辑会在这里执行。 其优势在于，您可以将授权策略应用于非MVC端点，以前必须以手动，强制性方式对其进行处理。

最后，通过调用`endpoints.MapControllers()`映射API控制器。 **它仅映射有路由特性的控制器-并不配置任何常规路由**。

## ASP.NET Core Web App (MVC) template

MVC模板项目（通过` dotnet new mvc `创建）包含的内容比Web API模板多一些，但与2.x版本中的内容相比，它有所减少。 只有[一个控制器](https://github.com/aspnet/AspNetCore/blob/v3.0.0-preview9.19424.4/src/ProjectTemplates/Web.ProjectTemplates/content/StarterWeb-CSharp/Controllers/HomeController.cs)，即HomeController，关联的Views和所需的共享Razor模板。

 *Startup.cs*非常类似于Web API模板，只有几个差异，代码如下：

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllersWithViews();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        else
        {
            app.UseExceptionHandler("/Home/Error");
            app.UseHsts();
        }

        app.UseHttpsRedirection();
        app.UseStaticFiles();

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllerRoute(
                name: "default",
                pattern: "{controller=Home}/{action=Index}/{id?}");
        });
    }
}
```

使用`AddControllersWithViews()`代替了`AddControllers()`扩展方法。 正如你所料，它将添加Web API和MVC通用的MVC Controller服务，而且还添加[渲染Razor视图所需的服务](https://github.com/aspnet/AspNetCore/blob/0303c9e90b5b48b309a78c2ec9911db1812e6bf3/src/Mvc/Mvc/src/MvcServiceCollectionExtensions.cs#L211)。

由于这是一个MVC应用程序，因此中间件还添加了除Development环境外的异常处理程序中间件，并且还添加了与2.2相同的HSTS和HTTPS重定向中间件。

接下来是静态文件中间件，它位于路由中间件之前。 这样可以确保无需为每个静态文件请求进行路由解析，这在MVC应用程序中可能很常见。 

与Web API模板的唯一不同之处在于，端点路由中间件中MVC控制器的注册内容。 在这种情况下，为MVC控制器添加了常规路由，而不是Web API常用的特性路由方式。 同样，这类似于2.x中的设置，但针对端点路由系统进行了调整。

## ASP.NET Core Web App (Razor) template

Razor Pages是ASP.NET Core 2.0中引入的，它是MVC的页面的一种替代方式。 对于应用程序，[Razor Pages提供了比MVC更为平易近人的模型](https://www.twilio.com/blog/introduction-asp-net-core-razor-pages)，它构建于是MVC基础之上，因此使用` dotnet new webapp `创建的*Startup.cs*看起来与MVC版本非常相似：

```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddRazorPages();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        else
        {
            app.UseExceptionHandler("/Error");
            app.UseHsts();
        }

        app.UseHttpsRedirection();
        app.UseStaticFiles();

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapRazorPages();
        });
    }
}
```

上面代码中第一个变化是将`AddControllersWithViews()`替换为`AddRazorPages()`。 如你所料，这将添加[Razor Pages所需的所有其他服务](https://github.com/aspnet/AspNetCore/blob/0303c9e90b5b48b309a78c2ec9911db1812e6bf3/src/Mvc/Mvc/src/MvcServiceCollectionExtensions.cs#L294)。 **有趣的是，它没有添加标准的MVC控制器与Razor Views所需的服务**。 如果要在应用程序中同时使用MVC和Razor Pages，则应改为使用`AddMvc()`扩展方法。

对*Startup.cs*的唯一其他更改是将MVC端点替换为Razor Pages端点。 与服务一样，如果希望在应用中同时使用MVC和Razor页面，则需要同时映射两个端点。例如：

```csharp
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers(); // 映射特性方式的路由
    endpoints.MapDefaultControllerRoute(); // 映射常规方式的Mvc Controller路由
    endpoints.MapRazorPages();
});
```

## 总结

这篇文章简要概述了使用不同ASP.NET Core模板创建的*Startup.cs*文件。 每个模板都在前一个模板基础上增加了一些额外的功能。 在许多方面，与.NET Core 2.x的模板非常相似。 

最大的新功能是能够更方便地创建，仅包含你所需要的最少功能的应用程序。 包含您的应用程序所需的MVC服务数量最少的功能以及新的端点路由。

