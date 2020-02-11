---
title: 将集成测试升级至.NET Core 3.0
tags: 
  - .NET CORE
  - .NET CORE 3.0
  - ASP.NET CORE
  - Testing

date: 2020-02-11 07:40:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Converting integration tests to .NET Core 3.0](https://andrewlock.net/converting-integration-tests-to-net-core-3/)
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [升级至 ASP.NET Core 3.0](/upgrading-to-asp-net-core-3) 第5篇:

1. [转换.NET Standard 2.0类库到.NET Core 3.0](/converting-a-netstandard-2-library-to-netcore-3/)
2. [对比IHostingEnvironment与IHostEnvironment .NET及Core 3.0中的过时类型](/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/)
3. [不要在Startup类的构造函数中使用依赖注入](/avoiding-startup-service-injection-in-asp-net-core-3/)
4. [将末端中间件转换为端点路由](/converting-a-terminal-middleware-to-endpoint-routing-in-aspnetcore-3/)
5. [将集成测试升级至.NET Core 3.0](/converting-integration-tests-to-net-core-3/)

在本文中，我们来讨论一下，当升级到ASP.NET Core 3.0后，集成测试代码中`WebApplicationFactory<>`或`TestServer`的变化。

ASP.NET Core 3.0的最大变化之一，是在通用主机架构上运行，而不是在WebHost上。 在本系列的前几篇文章及[探索ASP.NET Core 3.0的系列](https://andrewlock.net/series/exploring-asp-net-core-3/)中，我们已经解决了一部分升级后带来的问题，我们来看看对周边的基础设施有哪些影响，例如用于集成测试的`TestServer`。

<!-- more -->

## 使用Test Host和TestServer来做集成测试

ASP.NET Core包含*Microsoft.AspNetCore.TestHost*库，它是一个可以在内存中运行的Web主机。 

> 这里面用了容易让人混淆的术语。内存中的主机和NuGet包通常被称为“ **TestHost**”，而在代码中使用的实际类是**TestServer**。 两者经常互换使用。

在ASP.NET Core 2.x中，可以将`IWebHostBuilder`实例作为参数，传递给`TestServer`构造函数来创建测试服务器：

```csharp
public class TestHost2ExampleTests
{
    [Fact]
    public async Task ShouldReturnHelloWorld()
    {
        // Build your "app"
        var webHostBuilder = new WebHostBuilder()
            .Configure(app => app.Run(async ctx => 
                    await ctx.Response.WriteAsync("Hello World!")
            ));

        // Configure the in-memory test server, and create an HttpClient for interacting with it
        var server = new TestServer(webHostBuilder);
        HttpClient client = server.CreateClient();

        // Send requests just as if you were going over the network
        var response = await client.GetAsync("/");

        response.EnsureSuccessStatusCode();
        var responseString = await response.Content.ReadAsStringAsync();
        Assert.Equal("Hello World!", responseString);
    }
}
```

在上面的示例中，我们创建了一个简单的`WebHostBuilder`，针对所有的Url请求，都返回“ Hello World！”。 然后，使用`TestServer`创建一个内存服务器：

```csharp
var server = new TestServer(webHostBuilder);
```

最后，我们创建一个`HttpClient`对象，发送HTTP请求发送到内存服务器。 

```csharp
var client = server.CreateClient();

var response = await client.GetAsync("/");
```

在.NET core 3.0中，写法还是类似的，但是由于迁移到通用主机，使用方法稍微复杂一些。

## .NET core 3.0中的TestServer

如果要升级.NET Core 2.x测试项目到.NET Core 3.0，编辑项目的*.csproj*，然后将`<TargetFramework>`元素更改为`netcoreapp3.0`。 接下来，将*Microsoft.AspNetCore.App*的`<PackageReference>`替换为`<FrameworkReference>`，并将其他软件包版本更新为`3.0.0`。

升级至.NET Core 3.0项目后，项目运行没有任何错误，并且可以通过测试。 但是，该代码中使用的是`WebHost`，而不是通用主机。 下面改为使用通用主机。

首先，使用HostBuilder替代WebHostBuilder：

```csharp
var hostBuilder = new HostBuilder();
```

`HostBuilder`中没有`Configure()`方法。改成先调用`ConfigureWebHost()`，然后在内部`IWebHostBuilder`对象上调用`Configure()`。

```csharp
var hostBuilder = new HostBuilder()
    .ConfigureWebHost(webHost => 
        webHost.Configure(app => app.Run(async ctx =>
                await ctx.Response.WriteAsync("Hello World!")
    )));
```

改完后，`TestServer`构造函数这里编译报错。

![](https://cdn.ibestread.com/img/testserver.png)

[`TestServer`构造函数参数类型是`IWebHostBuilder`](https://github.com/aspnet/AspNetCore/blob/master/src/Hosting/TestHost/src/TestServer.cs#L46)，在通用主机中只有`IHostBuilder`。 我花了一些时间才发现根本不用手动创建`TestServer`：

- 在`ConfigureWebHost`中调用`UseTestServer()`以添加`TestServer`实现。
- 在`IHostBuilder`上调用`StartAsync()`方法来构建和启动一个`IHost`实例。
- 在已启动的`IHost`实例上调用`GetTestClient()`获取`HttpClient`实例。

最终的代码如下：

```csharp
public class TestHost3ExampleTests
{
    [Fact]
    public async Task ShouldReturnHelloWorld()
    {
        var hostBuilder = new HostBuilder()
            .ConfigureWebHost(webHost =>
            {
                // Add TestServer
                webHost.UseTestServer();
                webHost.Configure(app => app.Run(async ctx => 
                    await ctx.Response.WriteAsync("Hello World!")));
            });

        // Build and start the IHost
        var host = await hostBuilder.StartAsync();

        // Create an HttpClient to send requests to the TestServer
        var client = host.GetTestClient();

        var response = await client.GetAsync("/");

        response.EnsureSuccessStatusCode();
        var responseString = await response.Content.ReadAsStringAsync();
        Assert.Equal("Hello World!", responseString);
    }
}
```

如果没有执行`UseTestServer()`，运行时会看到下面错误：

```bash
System.InvalidOperationException : Unable to resolve service for type 'Microsoft.AspNetCore.Hosting.Server.IServer' while attempting to activate 'Microsoft.AspNetCore.Hosting.GenericWebHostService'
```

## 使用WebApplicationFactory集成测试

像上面那样直接使用`TestServer`非常方便，但是对于实际项目的集成测试则不太方便。 [*Microsoft.AspNetCore.Mvc.Testing*](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.Testing)需要处理一些比较棘手问题，例如设置*ContentRoot*路径，将*.deps*文件复制到测试项目的*bin*文件夹等。使用`WebApplicationFactory<>`可以简化`TestServer`的创建过程。

[如何使用WebApplicationFactory<>的文档](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-3.0#customize-webapplicationfactory)，在.NET Core 3.0任然适用。 但是，ASP.NET Core 2.x升级到3.0时，还是需要一些调整。

## 在ASP.NET Core 2.x中使用WebApplicationFactory添加XUnit日志

假设你的程序是按下面步骤设置的：

- 通过`dotnet new webapp`创建的一个.NET Core Razor应用程序。
- 针对Razor应用程序项目创建了一个集成测试项目。

这个演示项目的源码在[Github](https://github.com/andrewlock/blog-examples/tree/master/updating-test-host-to-3-0)中。

您可以按照[文档的说明](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-3.0#basic-tests-with-the-default-webapplicationfactory)直接在测试中使用`WebApplicationFactory<>`类。 但有时候我们想自定义使用自定义的`WebApplicationFactory<>`，比如，[替换一些Mock服务](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-3.0#inject-mock-services)、[自动运行数据库迁移](https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-3.0#customize-webapplicationfactory)、自定义`IHostBuilder`。

使用[xUnit ITestOutputHelper](https://xunit.net/docs/capturing-output)，将日志输出至`ILogger`中，方便查看`TestServer`中的错误信息。 [Martin Costello](https://twitter.com/martin_costello)提供的Nuget包[*MartinCostello.Logging.XUnit*](https://www.nuget.org/packages/MartinCostello.Logging.XUnit/)比较方便做集成。代码如下：

```csharp
public class ExampleAppTestFixture : WebApplicationFactory<Program>
{
    // Must be set in each test
    public ITestOutputHelper Output { get; set; }

    protected override IWebHostBuilder CreateWebHostBuilder()
    {
        var builder = base.CreateWebHostBuilder();
        builder.ConfigureLogging(logging =>
        {
            logging.ClearProviders(); // Remove other loggers
            logging.AddXUnit(Output); // Use the ITestOutputHelper instance
        });

        return builder;
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        // Don't run IHostedServices when running as a test
        builder.ConfigureTestServices((services) =>
        {
            services.RemoveAll(typeof(IHostedService));
        });
    }
}
```

`ExampleAppTestFixture`做了2件事：

- 从容器中删除所有的`IHostedServices`（后台运行的服务），以便它们在集成测试期间不会运行。通常我们不希望，在测试时后台服务自动订阅或消费RabbitMQ/KafKa的消息。
- 使用`ITestOutputHelper`xUnit的日志与ASP.NET Core基础设施的`ILogger`挂接。

要在测试中使用`ExampleAppTestFixture`，必须在测试类上实现`IClassFixture<T>`接口，将`ExampleAppTestFixture`通过构造函数注入，并设置Output属性。

```csharp
public class HttpTests: IClassFixture<ExampleAppTestFixture>, IDisposable
{
    readonly ExampleAppTestFixture _fixture;
    readonly HttpClient _client;

    public HttpTests(ExampleAppTestFixture fixture, ITestOutputHelper output)
    {
        _fixture = fixture;
        fixture.Output = output;
        _client = fixture.CreateClient();
    }

    public void Dispose() => _fixture.Output = null;

    [Fact]
    public async Task CanCallApi()
    {
        var result = await _client.GetAsync("/");

        result.EnsureSuccessStatusCode();

        var content = await result.Content.ReadAsStringAsync();
        Assert.Contains("Welcome", content);
    }
}
```

示例中请求RazorPages应用程序的主页，并在*body*中查找字符串“ Welcome”（位于<h1>标记中）。 应用程序生成的日志都通过管道传递到xUnit的输出，这使得易于理解集成测试失败时发生的原因：

```bash
[2019-10-29 18:33:23Z] info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
[2019-10-29 18:33:23Z] info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
...
[2019-10-29 18:33:23Z] info: Microsoft.AspNetCore.Routing.EndpointMiddleware[1]
      Executed endpoint '/Index'
[2019-10-29 18:33:23Z] info: Microsoft.AspNetCore.Hosting.Diagnostics[2]
      Request finished in 182.4109ms 200 text/html; charset=utf-8
```

## 在ASP.NET Core 3.0中使用WebApplicationFactory

将集成测试项目转换为目标.NET Core 3.0之后，似乎不需要进行任何更改。 但是要注意，`ExampleAppTestFixture`的`CreateWebHostBuilder()`方法永远不会被调用。

原因是`WebApplicationFactory`需要同时支持Web主机和通用主机。 如果在*Program.cs*中使用了`WebHostBuilder`，则工厂将调用`CreateWebHostBuilder()`并运行重写的方法。 但是，如果使用的是通用`HostBuilder`，则工厂将调用另一种方法`CreateHostBuilder()`。

修改*Factory*的代码，将`CreateWebHostBuilder`重命名为`CreateHostBuilder`，将返回类型从`IWebHostBuilder`更改为`IHostBuilder`，然后将调用更改为使用通用主机。 其他所有内容保持不变：

```csharp
public class ExampleAppTestFixture : WebApplicationFactory<Program>
{
    public ITestOutputHelper Output { get; set; }

    // Uses the generic host
    protected override IHostBuilder CreatHostBuilder()
    {
        var builder = base.CreateHostBuilder();
        builder.ConfigureLogging(logging =>
        {
            logging.ClearProviders(); // Remove other loggers
            logging.AddXUnit(Output); // Use the ITestOutputHelper instance
        });

        return builder;
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices((services) =>
        {
            services.RemoveAll(typeof(IHostedService));
        });
    }
}
```

> **请注意**，`ConfigureWebHost`方法不会更改-在两种情况下都会被调用，参数任然是`IWebHostBuilder`。

全部修改完成后，日志可以正常输出了。

## 总结

在这篇文章中，我描述了将应用程序从ASP.NET Core 2.1迁移到ASP.NET Core 3.0之后，集成测试中所需的一些更改。 仅且使用通用主机时，才需要进行这些更改。 如果要迁移到使用通用主机，则需要修改`TestServer`或`WebApplicationFactory`的代码。

修改`TestServer`代码，在`HostBuilder.ConfigureWebHost()`方法内调用`UseTestServer()`。 然后构建主机，并调用`StartAsync()`启动主机。 最后，调用`IHost.GetTestClient()`创建一个`HttpClient`。

修改自定义的`WebApplicationFactory`代码，如果使用`WebHost`，需要重写`CreateWebHostBuilder`方法。使用通用`Host`，需要重写CreateWebHostBuilder方法。