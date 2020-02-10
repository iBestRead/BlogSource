---
title: 不要在Startup类的构造函数中使用依赖注入
tags: 
  - .NET CORE
  - .NET CORE 3.0
  - Startup

date: 2020-02-08 09:41:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Avoiding Startup service injection in ASP.NET Core 3](https://andrewlock.net/avoiding-startup-service-injection-in-asp-net-core-3/)
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [升级至 ASP.NET Core 3.0](/upgrading-to-asp-net-core-3) 第3篇:

1. [转换.NET Standard 2.0类库到.NET Core 3.0](/converting-a-netstandard-2-library-to-netcore-3/)
2. [对比IHostingEnvironment与IHostEnvironment .NET及Core 3.0中的过时类型](/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/)
3. [不要在Startup类的构造函数中使用依赖注入](/avoiding-startup-service-injection-in-asp-net-core-3/)
4. [将末端中间件转换为端点路由](/converting-a-terminal-middleware-to-endpoint-routing-in-aspnetcore-3/)
5. [将集成测试升级至.NET Core 3.0](/converting-integration-tests-to-net-core-3/)

当从ASP.NET Core 2.x应用程序升级至.NET Core 3时，`Startup`中的代码需要有所改动，不能再随便将服务注入到`Startup`构造函数中了。

<!-- more -->

## 迁移到ASP.NET Core 3.0的通用主机

在.NET Core 3.0中，[开发团队对ASP.NET Core 3.0主机的基础架构进行了重构](/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/#IHostingEnvironment-IHostEnvironment-IWebHostEnviornment3者对比)。 到目前为止，我已经顺利升级了几个应用程序到ASP.Net Core 3.0。[ 迁移指南文档](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30)可以很好地引导您完成所需的步骤，我强烈建议您按照此文档来升级。

大多数情况下，只需要解决两个问题：

- 在ASP.NET Core 3.0中使用端点路由中间件
- 使用通用主机时，不要在`Startup`中注入服务。

第一点在很多文章中都有所提到。 端点路由是在ASP.NET Core 2.2中引入的，但仅限于MVC中使用。 在ASP.NET Core 3.0中，优先推荐使用端点路由，它有很多优点。

> 中间件的顺序对端点路由来说非常关键。 建议您在升级应用程序时，仔细阅读[迁移文档的这一部分](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30#routing-startup-code)。 在下面的文章中，我将展示如何将终端中间件转换为端点路由。

第二点，提到了不要在`Startup`注入服务的问题。 下面的文章，我将说明这种用法的问题以及如何解决。

## 在ASP.NET Core 2.x的Startup中注入服务

在ASP.NET core 2.x中，可以在*Program.cs*中将配置对象（Option或Setting）注入至容器，然后将配置对象注入到*Startup.cs*中使用。 通常都会使用这种方法来设置强类型的配置对象，方便其它类注入使用。

我们来看看ASP.NET core 2.x中的例子：

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseStartup<Startup>()
            .ConfigureSettings(); // <- Configure services we'll inject into Startup later
}
```

`CreateWebHostBuilder`中调用了`ConfigureSettings()`方法。 代码如下：

```csharp
public static class SettingsinstallerExtensions
{
    public static IWebHostBuilder ConfigureSettings(this IWebHostBuilder builder)
    {
        return builder.ConfigureServices((context, services) =>
        {
            var config = context.Configuration;

            services.Configure<ConnectionStrings>(config.GetSection("ConnectionStrings"));
            services.AddSingleton<ConnectionStrings>(
                ctx => ctx.GetService<IOptions<ConnectionStrings>>().Value)
        });
    }
}
```

这个方法内部通过`IWebHostBuilder`实例的`ConfigureServices()`方法，对配置对象进行了注入。所以，可以在`Startup`构造函数中获取到配置对象：

```csharp
public static class Startup
{
    public class Startup
    {
        public Startup(
            IConfiguration configuration, 
            ConnectionStrings ConnectionStrings) // Inject pre-configured service
        {
            Configuration = configuration;
            ConnectionStrings = ConnectionStrings;
        }

        public IConfiguration Configuration { get; }
        public ConnectionStrings ConnectionStrings { get; }

        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers();

            // Use ConnectionStrings in configuration
            services.AddDbContext<BloggingContext>(options =>
                options.UseSqlServer(ConnectionStrings.BloggingDatabase));
        }

        public void Configure(IApplicationBuilder app)
        {

        }
    }
}
```

通过使用强类型的配置对象，来设置（初始化）其他服务时，这是很常见的。 在上面的示例中，`ConnectionStrings`是强类型的配置对象，可以在[启动时进行校验]( https://andrewlock.net/adding-validation-to-strongly-typed-configuration-objects-in-asp-net-core/ )，以确保它们不为null。 

 但是在ASP.NET Core 3.0的通用主机中使用这种方式，则会出现下面的异常： 

```bash
Unhandled exception. System.InvalidOperationException: Unable to resolve service for type 'ExampleProject.ConnectionStrings' while attempting to activate 'ExampleProject.Startup'.
   at Microsoft.Extensions.DependencyInjection.ActivatorUtilities.ConstructorMatcher.CreateInstance(IServiceProvider provider)
   at Microsoft.Extensions.DependencyInjection.ActivatorUtilities.CreateInstance(IServiceProvider provider, Type instanceType, Object[] parameters)
   at Microsoft.AspNetCore.Hosting.GenericWebHostBuilder.UseStartup(Type startupType, HostBuilderContext context, IServiceCollection services)
   at Microsoft.AspNetCore.Hosting.GenericWebHostBuilder.<>c__DisplayClass12_0.<UseStartup>b__0(HostBuilderContext context, IServiceCollection services)
   at Microsoft.Extensions.Hosting.HostBuilder.CreateServiceProvider()
   at Microsoft.Extensions.Hosting.HostBuilder.Build()
   at ExampleProject.Program.Main(String[] args) in C:\repos\ExampleProject\Program.cs:line 21
```

出现异常的原因是，ASP.NET Core 3.0不再支持这种方式。 `Startup`构造函数中只能注入`IHostEnvironment`和`IConfiguration`。 

> 请注意，如果在ASP.NET Core 3.0中使用的是IWebHostBuilder而不是通用主机，是可以继续使用此方式。但是强烈建议尽可能修改代码使用新方法！

## 会出现2个单例对象

不要将其它服务注入`Startup`的原因是，它会创建两次*ServiceProvider*。 在上面的示例中，应用程序启动时需要一个`ConnectionStrings`配置对象，然而这个配置对象的唯一创建方式是，`ConfigureSettings()`内的`IServiceProvider`。

其实`ConfigureSettings()`中的`IServiceProvider`只是一个临时的*ServiceProvider*。 它创建`ConnectionStrings`并将其注入`Startup`；接着再注入其它的依赖服务和配置对象，这些服务和配置被作为*ConfigureServices*方法的参数执行；再然后，临时*ServiceProvider*被释放。 最后又重新创建一个新的*ServiceProvider*，这个*ServiceProvider*才是应用程序整个生命周期中的永久对象。

即便将服务生命周期配置为`Singleton`，也会被创建两次：

- 一次在临时*ServiceProvider*，被注入到了`Startup`类中
- 一次在永久*ServiceProvider*，被注入到应用程序中

对于上面这种的情况（只使用了一个配置对象），即便*ServiceProvider*被创建两次，也没有什么大问题。但是，我们的应用程序通常会有很多个配置对象。那么，就会出现很多个被“泄露”的配置对象的实例。 ASP.NET Core 3.0中的通用主机，针对此处的重构目的，减少泄露，使程序更加安全地运行。

## ConfigureServices需要注入的服务怎么办

不能再在`Startup`中注入其它服务（`IHostEnvironment`和`IConfiguration`除外）了。如果非要使用其它服务，又应该如何处理呢？下面会演示在`Startup.ConfigureServices`中，根据不同配置，注入不同服务到容器中：

```csharp
public class Startup
{
    public Startup(IdentitySettings identitySettings)
    {
        IdentitySettings = identitySettings;
    }

    public IdentitySettings IdentitySettings { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        if(IdentitySettings.UseFakeIdentity)
        {
            services.AddScoped<IIdentityService, FakeIdentityService>();
        }
        else
        {
            services.AddScoped<IIdentityService, RealIdentityService>();
        }
    }

    public void Configure(IApplicationBuilder app)
    {
        // ...
    }
}
```

示例中，根据注入的`IdentitySettings`的`UseFakeIdentity`的属性值，确定要注入哪个`IIdentityService`接口的实现，到底是Fake服务还是Real服务。

通过工厂方式注册，可以满足这个需求，并且与通用主机模式兼容。 例如：

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
        // configure the IdentitySettings for the DI container
        services.Configure<IdentitySettings>(Configuration.GetSection("Identity")); 

        // Register the implementations using their implementation name
        services.AddScoped<FakeIdentityService>();
        services.AddScoped<RealIdentityService>();

        // Retrieve the IdentitySettings at runtime, and return the correct implementation
        services.AddScoped<IIdentityService>(ctx => 
        {
            var identitySettings = ctx.GetRequiredService<IdentitySettings>();
            return identitySettings.UseFakeIdentity
                ? ctx.GetRequiredService<FakeIdentityService>()
                : ctx.GetRequiredService<RealIdentityService>();
            }
        });
    }

    public void Configure(IApplicationBuilder app)
    {
        // ...
    }
}
```

这种方法明显比之前的写法复杂得多，好处是至少与通用主机模式兼容了。

如果仅需要强类型配置（如本例所示），用此方法有些复杂。 我们可以使用更为简便的**重新绑定**（rebind）方式：

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
        // configure the IdentitySettings for the DI container
        services.Configure<IdentitySettings>(Configuration.GetSection("Identity")); 

        // "recreate" the strongly typed settings and manually bind them
        var identitySettings = new IdentitySettings();
        Configuration.GetSection("Identity").Bind(identitySettings)

        // conditionally register the correct service
        if(identitySettings.UseFakeIdentity)
        {
            services.AddScoped<IIdentityService, FakeIdentityService>();
        }
        else
        {
            services.AddScoped<IIdentityService, RealIdentityService>();
        }
    }

    public void Configure(IApplicationBuilder app)
    {
        // ...
    }
}
```

如果只是获取配置中某个字符串的话。 可以直接从`IConfiguration`实例中获取配置，就像ASP.NET Core默认模板项目中获取数据库连接字符串那种：

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
        // configure the ConnectionStrings for the DI container
        services.Configure<ConnectionStrings>(Configuration.GetSection("ConnectionStrings")); 

        // directly retrieve setting instead of using strongly-typed options
        var connectionString = Configuration["ConnectionString:BloggingDatabase"];

        services.AddDbContext<ApplicationDbContext>(options =>
                options.UseSqlite(connectionString));
    }

    public void Configure(IApplicationBuilder app)
    {
        // ...
    }
}
```

以上示例，虽然不是最好的解决方法，但是它们可以解决问题，在大多数情况下，都可以满足你的需求。 

假如，有个配置对象B依赖另外一个配置对象A。 对于这些情况，使用`IConfigureOptions`是一种更好的方法。

## 使用IConfigureOptions来配置选型

我们经常会使用注入`ConfigureOption`方式，比如使用`IdentityServer`时，[如其文档中所述](http://docs.identityserver.io/en/latest/topics/apis.html#the-identityserver-authentication-handler)：

```csharp
public class Startup
{
    public Startup(IdentitySettings identitySettings)
    {
        IdentitySettings = identitySettings;
    }

    public IdentitySettings IdentitySettings { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        // Configure IdentityServer Auth
        services
            .AddAuthentication(IdentityServerAuthenticationDefaults.AuthenticationScheme)
            .AddIdentityServerAuthentication(options =>
            {
                // Configure the authentication handler settings using strongly typed options
                options.Authority = identitySettings.ServerFullPath;
                options.ApiName = identitySettings.ApiName;
            });
    }

    public void Configure(IApplicationBuilder app)
    {
        // ...
    }
}
```

利用已注入的强类型配置对象`IdentitySettings`的实例，去赋值`IdentityServer.Option`属性，如Url和API资源名称。 这种设置方式在.NET Core 3.0中是不起作用，我们需要一种替代方式。 我们可以像上面示例那样，重新绑定强类型配置。 又或者，可以直接使用`IConfiguration`对象来获取配置。

源码中，`AddIdentityServerAuthentication()`先配置了JWT方式的身份验证，再指定的身份验证方案（IdentityServerAuthenticationDefaults.AuthenticationScheme），并对方案所需的强类型配置进行赋值。 这样，程序可以根据实际情况，使用` IConfigureOptions `实例的值，替换掉强类型配置的值。

在`IConfigureOptions`接口的实现中，可以注入其它服务来对此接口中的配置进行延迟初始化或赋值。 例如，要配置强类型的`TestSettings`，并需要调用`TestService`的某个方法来初始化，则可以创建一个`IConfigureOptions`实现，如下所示：

```csharp
public class MyTestSettingsConfigureOptions : IConfigureOptions<TestSettings>
{
    private readonly TestService _testService;
    public MyTestSettingsConfigureOptions(TestService testService)
    {
        _testService = testService;
    }

    public void Configure(TestSettings options)
    {
        options.MyTestValue = _testService.GetValue();
    }
}
```

在`Startup.ConfigureServices`方法中，同时注入`TestService`和`IConfigureOptions<TestSettings>`：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddScoped<TestService>();
    services.ConfigureOptions<MyTestSettingsConfigureOptions>();
}
```

如果采用这种方式，我们应该如何来配置IdentityServer？

首先，我们要了解一下，可命名Options（我之前已经讨论过几次，[链接1](https://andrewlock.net/configuring-named-options-using-iconfigurenamedoptions-and-configureall/)，[链接2](https://andrewlock.net/creating-singleton-named-options-with-ioptionsmonitor/)，[链接3](https://andrewlock.net/using-multiple-instances-of-strongly-typed-settings-with-named-options-in-net-core-2-x/)）。 `AddIdentityServerAuthentication`使用的就是Options。 

对可命名Options`IdentityServerAuthenticationOptions`进行延迟配置，真正的值是从强类型配置对象`IdentitySettings`中获取。 创建一个将`IdentitySettings`作为构造函数参数的`ConfigureIdentityServerOptions`对象：

```csharp
public class ConfigureIdentityServerOptions : IConfigureNamedOptions<IdentityServerAuthenticationOptions>
{
    readonly IdentitySettings _identitySettings;
    public ConfigureIdentityServerOptions(IdentitySettings identitySettings)
    {
        _identitySettings = identitySettings;
        _hostingEnvironment = hostingEnvironment;
    }

    public void Configure(string name, IdentityServerAuthenticationOptions options)
    { 
        // Only configure the options if this is the correct instance
        if (name == IdentityServerAuthenticationDefaults.AuthenticationScheme)
        {
            // Use the values from strongly-typed IdentitySettings object
            options.Authority = _identitySettings.ServerFullPath; 
            options.ApiName = _identitySettings.ApiName;
        }
    }

    // This won't be called, but is required for the IConfigureNamedOptions interface
    public void Configure(IdentityServerAuthenticationOptions options) => Configure(Options.DefaultName, options);
}
```

在*Startup.cs*中，配置强类型的`IdentitySettings`对象，添加所需的`IdentityServer`服务，并注册`ConfigureIdentityServerOptions`类，以便它可以在需要时，配置`IdentityServerAuthenticationOptions`对象：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Configure strongly-typed IdentitySettings object
    services.Configure<IdentitySettings>(Configuration.GetSection("Identity"));

    // Configure IdentityServer Auth
    services
        .AddAuthentication(IdentityServerAuthenticationDefaults.AuthenticationScheme)
        .AddIdentityServerAuthentication();

    // Add the extra configuration;
    services.ConfigureOptions<ConfigureIdentityServerOptions>();
}
```

## 总结

在本文中，我描述了升级到ASP.NET Core 3.0时，*Startup.cs*中的一些更改。 描述在ASP.NET Core 2.x中，将服务注入到`Startup`时遇到的问题，以及如何在ASP.NET Core 3.0中的替换方式。 