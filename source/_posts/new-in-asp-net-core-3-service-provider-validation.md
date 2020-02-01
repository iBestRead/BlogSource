---
title: ASP.Net Core 3.0的新特性-Service provider validation
tags: 
  - dotnet-core
  - asp-dotnet-core
  - asp-dotnet-core-3
  - service provider
  - validation
  - DI
  - IOC

date: 2020-02-01 14:39:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [New in ASP.NET Core 3: Service provider validation](https://andrewlock.net/new-in-asp-net-core-3-service-provider-validation/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [探索 ASP.NET Core 3.0](https://blog.ibestread.com/exploring-asp-net-core-3) 第3篇:

1. [ASP.Net Core 3.0中的.csproj文件,Program.cs及通用主机](exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/)
2. [ASP.Net Core 3.0的Startup.cs在不同项目类型中的差异](https://blog.ibestread.com/comparing-startup-between-the-asp-net-core-3-templates/)
3. ASP.Net Core 3.0的新特性-Service provider validation(本文)
4. [ASP.Net Core 3.0中应用程序启动时运行异步任务](https://blog.ibestread.com/running-async-tasks-on-app-startup-in-asp-net-core-3)
5. [介绍IHostLifetime及与通用主机间的作用关系](https://blog.ibestread.com/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions)
6. [ASP.Net Core 3.0的新特性-启动时的结构化日志](https://blog.ibestread.com/new-in-aspnetcore-3-structured-logging-for-startup-messages)
7. [ASP.Net Core 3.0的新特性-本地工具](https://blog.ibestread.com/new-in-net-core-3-local-tools)

此篇文章来介绍ASP.NET Core 3.0中的新功能“编译时验证”。 此功能可以用来检测依赖注入中的配置错误。 具体来说，就是检查未在容器中注入但被依赖的服务。

首先，将展示该功能的工作原理，然后再介绍一些陷阱，当依赖注入容器有配置错误的时候，“编译时验证”并没有检查到这些问题。

> 需要指出的是：检查依赖注入配置并不是一个全新想法，这只是我们经常使用的一个功能`StructureMap`，参考[Lamar](https://jasperfx.github.io/lamar/documentation/ioc/diagnostics/validating-container-configuration/)

 <!-- more --> 

## 简单的示例

在这篇文章中，使用` dotnet new webapi `来创建一个应用程序。 它包含`WeatherForecastService`控制器，该控制器会返回一些随机数据。

首先，我们先重构一下控制器：

```csharp
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase
{
    private readonly WeatherForecastService _service;
    public WeatherForecastController(WeatherForecastService service)
    {
        _service = service;
    }

    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        return _service.GetForecasts();
    }
}
```

上面的控制器依赖于`WeatherForecastService`。代码如下：

```csharp
public class WeatherForecastService
{
    private readonly DataService _dataService;
    public WeatherForecastService(DataService dataService)
    {
        _dataService = dataService;
    }

    public IEnumerable<WeatherForecast> GetForecasts()
    {
        var data = _dataService.GetData();

        // use data to create forcasts

        return new List<WeatherForecast>();
    }
}
```

这个服务又依赖于`DataService`，代码如下：

```csharp
public class DataService
{
    public string[] GetData() => new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };
}
```

以上就是我们需要的所有服务，然后我们在` Startup.ConfigureServices `方法中，注入它们：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    services.AddSingleton<WeatherForecastService>();
    services.AddSingleton<DataService>();
}
```

在这个示例中，我们把服务全部注册为单例。在本章节中服务的生命周期（单例，瞬时，范围等）并不是要讲述的重点。运行程序，并访问` /WeatherForecast `，我们会得到下面结果：

```json
[{
    "date":"2019-09-07T22:29:31.5545422+00:00",
    "temperatureC":31,
    "temperatureF":87,
    "summary":"Sweltering"
}]
```

程序现在是正常的。如果我们故意不注入某个服务，看看会发生什么？

## 在启动时检查未注入的依赖

我们故意遗漏 `DataService` 的注入，代码如下：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    services.AddSingleton<WeatherForecastService>();
    // services.AddSingleton<DataService>();
}
```

当我们再次运行，会看到一个异常，包含异常的堆栈信息，提示程序无法启动。下面是该异常的部分信息：

```bash
Unhandled exception. System.AggregateException: Some services are not able to be constructed
(Error while validating the service descriptor 
    'ServiceType: TestApp.WeatherForecastService Lifetime: Scoped ImplementationType:
     TestApp.WeatherForecastService': Unable to resolve service for type
    'TestApp.DataService' while attempting to activate 'TestApp.WeatherForecastService'.) 
```

这个异常的错误信息非常明确：“ Unable to resolve service for type
    'TestApp.DataService' while attempting to activate 'TestApp.WeatherForecastService' ” ，意思是：在试图创建'TestApp.WeatherForecastService'实例时，无法解析类型为'TestApp.DataService'的服务。可以看到依赖注入验证起了作用。它有助于我们，减少程序启动时因为依赖注入配置问题而产生的错误。但是相对于编译时就提示错误，这种运行时才抛出的异常就显的不那么方便了。

如果我们遗漏`WeatherForecastService`注入又会发生什么？

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();
    // services.AddSingleton<WeatherForecastService>();
    services.AddSingleton<DataService>();
}
```

这种情况下，程序启动是正常的。但是当访问API时，就会发生异常！

## 陷阱1：控制器的构造函数不会被依赖注入验证器检查

验证器未能检查到这个问题的原因是：所有的控制器并不是由依赖注入容器创建的。正如[之前的文章](https://andrewlock.net/controller-activation-and-dependency-injection-in-asp-net-core-mvc/)所说， `DefaultControllerActivator` 只是从容器中获取了服务间的依赖关系，而并未使用容器来创建。因此，容器中没有控制器，所以就无法检查控制器的依赖项是否已经注册。

幸运的是有方法来解决。我们可以通过 `AddControllersAsServices() `方法将所有的控制器都作为服务添加到容器中：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers()
        .AddControllersAsServices(); // Add the controllers to DI

    // services.AddSingleton<WeatherForecastService>();
    services.AddSingleton<DataService>();
}
```

通过这种方式启用`ServiceBasedControllerActivator`（[查看之前的文章](https://andrewlock.net/controller-activation-and-dependency-injection-in-asp-net-core-mvc/#the-servicebasedcontrolleractivator)）并将控制器作为服务注入。再次运行程序，就可以看到因为缺少控制器的依赖而引发的异常：

```bash
Unhandled exception. System.AggregateException: Some services are not able to be constructed
(Error while validating the service descriptor 
    'ServiceType: TestApp.Controllers.WeatherForecastController Lifetime: Transient
    ImplementationType: TestApp.Controllers.WeatherForecastController': Unable to 
    resolve service for type 'TestApp.WeatherForecastService' while attempting to
    activate'TestApp.Controllers.WeatherForecastController'.)
```

似乎这是一个简便的解决方法。但是不确定这是不是我想要的，毕竟它解决了问题。

另外构造函数注入并不是依赖注入唯一方式，我们还要看看其他注入方式。

## 陷阱2：[FromServices]方式不会被依赖注入验证器检查

有时候我们也会使用[模型绑定来创建MVC Action的方法参数](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/model-binding?view=aspnetcore-2.2)，例如:  常用的attributes方式`[FromBody]`或`FromQuery`。

同样，也可以将`[FromServices]`属性应用于Action的方法参数，通过从依赖注入容器来创建这些参数。 此功能对某个服务只被单个Action方法所依赖时非常有用，可以避免将此服务注入到整个控制器中，其他Action方法并不依赖此服务。

例如，我们可以使用`[FromServices]`来注入`WeatherForecastController`： 

```csharp
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase
{
    [HttpGet]
    public IEnumerable<WeatherForecast> Get(
        [FromServices] WeatherForecastService service) // injected using DI
    {
        return service.GetForecasts();
    }
}
```

通过这种方式，依赖注入验证器也是无法检查到的。程序运行是正常的，但是访问API时就会发生异常。

要规避这个问题，最简单的方式就是只使用构造函数注入，而不要使用`[FromServices]`。

还有另外一种方式：通过`IServiceProvider`直接解析。

## 陷阱3：通过IServiceProvider方式不会被依赖注入验证器检查

我们再来改写下`WeatherForecastController`。这次不是直接注入`WeatherForecastService`，我们先注入一个的`IServiceProvider`，再用它直接解析一个服务（[这是一种反模式](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)）。 

```csharp
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase
{
    private readonly WeatherForecastService _service;
    public WeatherForecastController(IServiceProvider provider)
    {
        _service = provider.GetRequiredService<WeatherForecastService>();
    }

    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        return _service.GetForecasts();
    }
}
```

使用`IServiceProvider`这种方式，并不是一个好主意。因为它没有明确表示控制器`WeatherForecastController`对其他服务的依赖关系，隐藏了对`WeatherForecastService`的依赖。除了开发人员难以理解之外，验证器也解析不了依赖关系。程序可以正常运行，但是访问API时就会发生异常。

不幸的是，在有些情况下是一定要用到`IServiceProvider`的。 例如，有一个单例对象，该对象依赖`scoped`服务，[之前文章有讲过](https://andrewlock.net/the-dangers-and-gotchas-of-using-scoped-services-when-configuring-options-in-asp-net-core/#3-creating-a-new-scope-in-iconfigureoptions)  。再或者有一个单例对象，该对象不能具有构造函数方式的依赖注入，例如[验证attributes](https://andrewlock.net/injecting-services-into-validationattributes-in-asp-net-core/)。 这些情况下验证器均无法验证。

当您使用工厂函数创建依赖项时，类似的陷阱就不太明显了。

## 陷阱4：通过工厂方式不会被依赖注入验证器检查

修改控制器，注入`WeatherForecastService`到构造函数，并使用`AddControllersAsServices()`方法。另外再做两个修改： 

1. 遗漏`DataService`的注入。
2. 使用工厂创建`WeatherForecastService`对象。

说到工厂方式，是指在服务注册时使用`lambda`表达式，返回需创建的服务实例。 例如：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers()
        .AddControllersAsServices();
    services.AddSingleton<WeatherForecastService>(provider => 
    {
        var dataService = new DataService();
        return new WeatherForecastService(dataService);
    });
    // services.AddSingleton<DataService>(); // not required

}
```

在上面的示例中，我们通过工厂方式使用`lambda`表达式，创建了一个`WeatherForecastService`，在`lambda`内部，我们手动实例化了`DataService`和`WeatherForecastService`。

使用这种方式，应用程序不会出现任何问题，因为我们可以使用工厂方式从依赖注入容器中解析到了`WeatherForecastService`。 我们手动创建了`WeatherForecastService`需要析`DataService`对象，而无需使用容器去解析它，因此程序没有问题。

再换一种写法：

```cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers()
        .AddControllersAsServices();
    services.AddSingleton<WeatherForecastService>(provider => 
    {
        var dataService = provider.GetRequiredService<DataService>();
        return new WeatherForecastService(dataService);
    });
    // services.AddSingleton<DataService>(); // Required!
}
```

此种写法是通过`IServiceProvider`在运行时解析`DataService`服务，这属于隐式依赖。跟上面的**陷阱3**是一样的。验证器是无法验证的。

与之前的陷阱一样，有些情况下，这样的代码是必需的，而且没有简单的方法来解决它。 如果遇到这种情况，请格外小心，以确保您请求的依赖项已正确注入。

## 陷阱5：开放性的泛型不验证

最后一个小问题，[ASP.NET Core 申明了对开放性的泛型不验证](https://github.com/aspnet/Extensions/blob/fb7fdf5550614f6d73c87f46cbcd0d2409a46095/src/DependencyInjection/DI/src/ServiceProviderOptions.cs#L23)。

例如，假设我们有一个通用的`ForcastService <T>`：

```csharp
public class ForecastService<T> where T: new();
{
    private readonly DataService _dataService;
    public ForecastService(DataService dataService)
    {
        _dataService = dataService;
    }

    public IEnumerable<T> GetForecasts()
    {
        var data = _dataService.GetData();

        // use data to create forcasts

        return new List<T>();
    }
}
```

 在`Startup.cs`我们注入了的开放性的通用泛型，再次遗漏了注册`DataService`： 

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers()
        AddControllersAsServices();

    // register the open generic
    services.AddSingleton(typeof(ForecastService<>));
    // services.AddSingleton<DataService>(); // should cause an error
}
```

*service provider*完全跳过了开放性泛型注册验证，因此永远不会检测到丢失的`DataService`依赖项。 应用程序运行时没有错误，但是在尝试请求`ForecastService <T>`时将引发运行时异常。

但是，如果应用程序中使用约束性泛型，那么验证器将检测到该问题。 例如，我们可以使用类型`WeatherForecast`约束泛型的`T`，作为`WeatherForecastController`的依赖项：

```csharp
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase
{
    private readonly ForecastService<WeatherForecast> _service;
    public WeatherForecastController(ForecastService<WeatherForecast> service)
    {
        _service = service;
    }

    [HttpGet]
    public IEnumerable<WeatherForecast> Get()
    {
        return _service.GetForecasts();
    }
}
```

没问题，*service provider*验证器确实检测到了！实际上，使用开发性泛型注入的方式，不像使用`IServiceProvider`或工厂方式方式那么重要。你可以采用约束性泛型注入来替代开放性泛型注入（除非该服务本身就是开放性的）。另外，如果使用`IServiceProvider`来解析开放性泛型，恭喜你，又回到了**陷阱3**和**陷阱4**中了。

## 在其他环境中开启验证器

这是我所知道的最后一个陷阱，需要重点关注。默认情况下，*service provider*验证仅在开发环境中启用了。 因为开启此功能会有额外开销，与 [scope validation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2#scope-validation) 是相同。

但是，如果你的代码中有“条件服务注册”，例如，在*Development*环境中注册的服务与在其他环境中注册的服务不同，并且还希望在其他环境中也开启*service provider*验证。 可以在*Program.cs*代码中调用`UseDefaultServiceProvider()`扩展方法实现。 在下面的示例中，我已在所有环境中启用`ValidateOnBuild`，但仅在开发中保留了*scope validation*：

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            })
            // Add a new service provider configuration
            .UseDefaultServiceProvider((context, options) =>
            {
                options.ValidateScopes = context.HostingEnvironment.IsDevelopment();
                options.ValidateOnBuild = true;
            });
}
```

## 总结

在这篇文章中，我描述了.NET Core 3.0中新增的`ValidateOnBuild`功能。 这使*Microsoft.Extensions* DI容器可以在首次编译*service provider*时就检查配置中的错误。 便于我们检测应用程序启动时的问题，而不是在运行时再暴露依赖注入的错误配置的问题。

虽然此功能很有用，但在一些情况下仍无法正常验证，例如，注入控制器、使用`IServiceProvider`解析或使用开放性泛型注入等。 这个功能并不能百分之百的解决你程序的依赖注入问题，有些情况下需要你自己解决。

