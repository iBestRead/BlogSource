---
title: 对比IHostingEnvironment与IHostEnvironment .NET及Core 3.0中的过时类型
tags: 
  - .NET CORE
  - .NET CORE 3.0
  - HostEnvironment

date: 2020-02-07 09:30:00
---

> 译者:  [Akini Xu](/)
>
> 原文:  [IHostingEnvironment vs IHostEnvironment - obsolete types in .NET Core 3.0](https://andrewlock.net/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [升级至 ASP.NET Core 3.0](/upgrading-to-asp-net-core-3) 第2篇:

1. [转换.NET Standard 2.0类库到.NET Core 3.0](/converting-a-netstandard-2-library-to-netcore-3/)
2. [对比IHostingEnvironment与IHostEnvironment .NET及Core 3.0中的过时类型](/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/)
3. [不要在Startup类的构造函数中使用依赖注入](/avoiding-startup-service-injection-in-asp-net-core-3/)
4. [将末端中间件转换为端点路由](/converting-a-terminal-middleware-to-endpoint-routing-in-aspnetcore-3/)
5. [将集成测试升级至.NET Core 3.0](/converting-integration-tests-to-net-core-3/)

在本文中，我将介绍.NET Core 3.0中已过时的类型与ASP.NET Core之间的差异。 说明它们变化的原因，并介绍什么时候及什么地方使用它们。

<!-- more -->

## ASP.NET Core与通用主机合并

ASP.NET Core 2.1引入了一个新对象通用主机（GenericHost），它可以用来构建非HTTP的应用程序。Microsoft.Extensions.\*的相关扩展方法可以用于应用程序配置，依赖项注入和日志记录等。 尽管这是一个非常不错的想法，但是在基础架构上，通用主机定义的一些抽象对象与Web主机（ASP.NET Core中使用）根本不兼容。 这导致了[名称空间冲突和不兼容](https://andrewlock.net/the-asp-net-core-generic-host-namespace-clashes-and-extension-methods/)，大多数情况下，我都避免使用通用主机。

在ASP.NET Core 3.0中，开发团队重构了Web主机的基础架构，以使其与通用主机兼容。 合并了之前的2个不同主机下的抽象对象（ASP.NET Core下的Web主机抽象对象与通用主机的抽象对象），[ASP.NET Core Web主机可以作为*IHostedService*](https://github.com/aspnet/AspNetCore/blob/1599a5a2a791def8a23784756c303a914c0e1631/src/Hosting/Hosting/src/GenericHost/GenericWebHostedService.cs)在通用主机之上运行。

从2.x升级到3.0时，ASP.NET Core 3不会强迫您立即转换为新的基于通用主机方式。 可以继续使用*WebHostBuilder*而不是*HostBuilder*。 [迁移文档](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30)上提示是必需的，但实际上，如果您出于某种原因希望继续使用它。

> 如果可能的话，我建议您升级使用HostBuilder。虽然现在WebHostBuilder未被标记为已过时（[Obsolete]），但是我认为在将来某个版本，它会被完全删除。

通用主机重构后，以前一些[重复定义的类型](https://andrewlock.net/the-asp-net-core-generic-host-namespace-clashes-and-extension-methods/)已被标记为过时，并且引入了新类型。 最好的例子是IHostingEnvironment。

## IHostingEnvironment,IHostEnvironment,IWebHostEnviornment3者对比

`IHostingEnvironment`是.NET Core 2.x中最令人讨厌的接口之一，因为它同时存在于*Microsoft.AspNetCore.Hosting*和*Microsoft.Extensions.Hosting*两个不同的命名空间中。 它们略有不同且互不兼容，一个不能从另一个继承。

```csharp
namespace Microsoft.AspNetCore.Hosting
{
    public interface IHostingEnvironment
    {
        string EnvironmentName { get; set; }
        string ApplicationName { get; set; }
        string WebRootPath { get; set; }
        IFileProvider WebRootFileProvider { get; set; }
        string ContentRootPath { get; set; }
        IFileProvider ContentRootFileProvider { get; set; }
    }
}

namespace Microsoft.Extensions.Hosting
{
    public interface IHostingEnvironment
    {
        string EnvironmentName { get; set; }
        string ApplicationName { get; set; }
        string ContentRootPath { get; set; }
        IFileProvider ContentRootFileProvider { get; set; }
    }
}
```

之所以存在2个接口的原因是，*Microsoft.AspNetCore.Hosting.IHostingEnvironment*早已存在，但*Microsoft.Extensions.Hosting.IHostingEnvironment*是随ASP.NET Core 2.1中的通用主机一起引入的。*Microsoft.Extensions.Hosting.IHostingEnvironment*没有关于静态文件`wwwroot`目录的相关属性（因为它用于承载非HTTP服务），因此它缺少`WebRootFileProvider`和`WebRootPath`属性。

出于向后兼容的原因，需要单独的抽象。 但是，这样做真正令人烦恼的结果是，无法编写同时适用于此接口2个版本的扩展方法。

在ASP.NET Core 3.0中，这两个接口都被标记为过时的。 您仍然可以使用它们，但是在编译时会收到警告。 同时引入了两个新接口：`IHostEnvironment`和`IWebHostEnvironment`。 尽管它们仍位于各自独立的命名空间下，但接口名词是不一样的，`IWebHostEnvironment`继承自`IHostEnvironment`。

```csharp
namespace Microsoft.Extensions.Hosting
{
    public interface IHostEnvironment
    {
        string EnvironmentName { get; set; }
        string ApplicationName { get; set; }
        string ContentRootPath { get; set; }
        IFileProvider ContentRootFileProvider { get; set; }
    }
}

namespace Microsoft.AspNetCore.Hosting
{
    public interface IWebHostEnvironment : IHostEnvironment
    {
        string WebRootPath { get; set; }
        IFileProvider WebRootFileProvider { get; set; }
    }
}
```

这种具有继承关系的层次更有意义，避免了重复，并且意味着针对通用主机版本（`IHostEnvironment`）的方法，现在在Web版本（`IWebHostEnvironment`）上任然适用。 本质上，`IHostEnvironment`和`IWebHostEnvironment`的实现是相同的，另外还实现了新接口。 例如，[ASP.NET Core实现](https://github.com/aspnet/AspNetCore/blob/v3.0.0/src/Hosting/Hosting/src/Internal/HostingEnvironment.cs)：

```csharp
namespace Microsoft.AspNetCore.Hosting
{
    internal class HostingEnvironment : IHostingEnvironment, Extensions.Hosting.IHostingEnvironment, IWebHostEnvironment
    {
        public string EnvironmentName { get; set; } = Extensions.Hosting.Environments.Production;
        public string ApplicationName { get; set; }
        public string WebRootPath { get; set; }
        public IFileProvider WebRootFileProvider { get; set; }
        public string ContentRootPath { get; set; }
        public IFileProvider ContentRootFileProvider { get; set; }
    }
}
```

那么应该使用哪个接口呢？ 答案是“**尽可能使用IHostEnvironment**”，但是其中还是由些差别……

**构建ASP.NET Core 3.0应用程序时**

尽可能使用`IHostEnvironment`，当需要访问`WebRootPath`或`WebRootFileProvider`属性时使用`IWebHostEnvironment`。

**构建被通用主机或.NET Core使用的类库项目时**

使用`IHostEnvironment`。 您的类库仍可与ASP.NET Core 3.0应用程序一起使用。

**构建被ASP.NET Core 3.0应用程序使用的类库项目时**

和之前一样，最好使用`IHostEnvironment`，因为您的类库可能会被其他通用宿主应用程序使用，而不仅仅针对ASP.NET Core应用程序。 但如果需要访问`IWebHostEnvironment`的其他属性，则必须将类库目标框架改为`netcoreapp3.0`而不是`netstandard2.0`，并添加一个<FrameworkReference>元素，如[之前的文章](/converting-a-netstandard-2-library-to-netcore-3/#依赖了ASP-NET-Core相关包)中所述。

**构建同时被ASP.NET Core 2.x和ASP.NET Core 3.0使用的类库项目时**

这种情况，有2种选择：

- 继续使用*Microsoft.AspNetCore*版本的`IHostingEnvironment`。 它可以在2.x和3.0中正常工作，如果将来的版本不支持了，再停止使用它即可。
- 使用条件编译`#if`，在*ASP.NET Core 3.0*中引入`IHostEnvironment`，在*ASP.NET Core 2.0*中引入`IHostingEnvironment`编译即可。

## IApplicationLifetime对比IHostApplicationLifetime

与前面的情况一样，`IApplicationLifetime`接口也存在冲突问题。 *Microsoft.Extensions.Hosting*和*Microsoft.AspNetCore.Hosting*中都存在此接口。 2个接口都是一样的：

```csharp
namespace Microsoft.AspNetCore.Hosting
{
    public interface IApplicationLifetime
    {
        CancellationToken ApplicationStarted { get; }
        CancellationToken ApplicationStopped { get; }
        CancellationToken ApplicationStopping { get; }
        void StopApplication();
    }
}

namespace Microsoft.Extensions.Hosting
{
    public interface IApplicationLifetime
    {
        CancellationToken ApplicationStarted { get; }
        CancellationToken ApplicationStopped { get; }
        CancellationToken ApplicationStopping { get; }
        void StopApplication();
    }
}
```

如您所料，这种重复是为了向后兼容。 .NET Core 3.0引入了一个新的接口`IHostApplicationLifetime`，该接口仅在*Microsoft.Extensions.Hosting*命名空间中定义，通用主机和ASP.NET Core应用程序中均可用：

```csharp
namespace Microsoft.Extensions.Hosting
{
    public interface IHostApplicationLifetime
    {
        CancellationToken ApplicationStarted { get; }
        CancellationToken ApplicationStopping { get; }
        CancellationToken ApplicationStopped { get; }
        void StopApplication();
    }
}
```

同样，此接口与以前的版本相同，.NET Core 3.0两个版本接口实现都为[`ApplicationLifetime`](https://github.com/aspnet/Extensions/blob/66007df6e5b68683de0cab0dc6a8552265318f20/src/Hosting/Hosting/src/Internal/ApplicationLifetime.cs)。 正如之前[有关启动过程的文章](/introducing-ihostlifetime-and-untangling-the-generic-host-startup-interactions/)中所讨论的那样，`ApplicationLifetime`对象在通用主机的启动和关闭过程中起着关键作用。 *Microsoft.AspNetCore.Hosting*中则没有与之等效的对象（通过扩展方法方式的有）。 [在AspNetCore中唯一的实现是一个简单的包装类型](https://github.com/aspnet/AspNetCore/blob/v3.0.0/src/Hosting/Hosting/src/GenericHost/GenericWebHostApplicationLifetime.cs)，该类型对`ApplicationLifetime`进行了简单封装，上相关委托都转发至通用主机的`IHostApplicationLifetime`接口实现上。

```csharp
namespace Microsoft.AspNetCore.Hosting
{
    internal class GenericWebHostApplicationLifetime : IApplicationLifetime
    {
        private readonly IHostApplicationLifetime _applicationLifetime;
        public GenericWebHostApplicationLifetime(IHostApplicationLifetime applicationLifetime)
        {
            _applicationLifetime = applicationLifetime;
        }

        public CancellationToken ApplicationStarted => _applicationLifetime.ApplicationStarted;
        public CancellationToken ApplicationStopping => _applicationLifetime.ApplicationStopping;
        public CancellationToken ApplicationStopped => _applicationLifetime.ApplicationStopped;
        public void StopApplication() => _applicationLifetime.StopApplication();
    }
}
```

应用程序生命周期的接口的选择，要比`IHostingEnvironment`的选择容易很多。

**构建.NET Core 3.0或ASP.NET Core 3.0应用程序或类库时**

使用`IHostApplicationLifetime`。 它仅需要引用*Microsoft.Extensions.Hosting.Abstractions*，并且可在所有应用程序中使用

**构建同时被ASP.NET Core 2.x和ASP.NET Core 3.0使用的类库项目时**

还有2中方法可以选择：

- 继续使用*Microsoft.Extensions*版本的`IApplicationLifetime`。 它可以在2.x和3.0中正常工作，如果将来的版本不支持了，再停止使用它即可。
- 使用条件编译`#if`，在*ASP.NET Core 3.0*中引入`IHostApplicationLifetime`，在*ASP.NET Core 2.0*中引入`IApplicationLifetime`编译即可。

幸运的是，与`IHostingEnvironment`相比，`IApplicationLifetime`的使用频率通常要低得多，因此使用`IApplicationLifetime`不会有太多困难。

## IWebHost对比IHost

`IWebHost`接口还没有继承自ASP.NET Core 3.0中的`IHost`。 同样，`IWebHostBuilder`也没有继承自`IHostBulider`。 它们仍然是完全独立的接口-一个用于ASP.NET Core，一个用于通用主机。

但是，没关系。 ASP.NET Core 3.0重构后使用了通用主机这个抽象类型，您可以使用通用主机`IHostBuilder`抽象类型的方法，并在ASP.NET Core和通用主机应用程序间共用。 如果需要执行只属于ASP.NET Core的操作，则仍可以使用`IWebHostBuilder`接口。

例如，考虑以下两种扩展方法，一种用于`IHostBuilder`，一种用于`IWebHostBuilder`：

```csharp
public static class ExampleExtensions
{
    public static IHostBuilder DoSomethingGeneric(this IHostBuilder builder)
    {
        // ... add generic host configuration
        return builder;
    }

    public static IWebHostBuilder DoSomethingWeb(this IWebHostBuilder builder)
    {
        // ... add web host configuration
        return builder;
    }
}
```

第一个方法在通用主机上进行某些配置（例如，它使用DI注册某些服务），第二个方法在`IWebHostBuilder`上进行某些配置。 例如，为Kestrel服务器设置一些默认值。

如果创建一个全新的ASP.NET Core 3.0应用程序，*Program.cs*代码如下：

```csharp
public class Program
{
    public static void Main(string[] args) => CreateHostBuilder(args).Build().Run();

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder
                    .UseStartup<Startup>();
            });
}
```

可以先对`IHostBuilder`调用`DoSomethingGeneric`扩展方法，再在`ConfigureWebHostDefaults()`内部的`IWebHostBuilder`调用`DoSomethingWeb`扩展方法，通过这种方式，可以同时对两种主机类型进行相关配置。

```csharp
public class Program
{
    public static void Main(string[] args) => CreateHostBuilder(args).Build().Run();

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .DoSomethingGeneric() // IHostBuilder extension method
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder
                    .DoSomethingWeb() // IWebHostBuilder extension method
                    .UseStartup<Startup>();
            });
}
```

可以在ASP.NET Core 3.0中，对两种Builder类型进行调用意味着，可以创建仅依赖于通用主机抽象类型的类库，并在ASP.NET Core应用中引用它们。

## 总结

在本文中，我讨论了ASP.NET Core 3.0中一些已过时的类型，它们的替代类型以及它们的过时原因。 如果要将应用程序升级为ASP.NET Core 3.0，则不必替换它们，因为它们提供的方法和之前的版本还是一样的。 但是在以后的版本中，这些过时类型会被移除，因此，进行更新还是有必要的。 