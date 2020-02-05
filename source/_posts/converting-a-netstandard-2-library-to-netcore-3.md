---
title: 转换.NET Standard 2.0类库到.NET Core 3.0
tags: 
  - .NET CORE
  - .NET CORE 3.0

date: 2020-02-05 22:01:00
---

> 译者:  [Akini Xu](https://blog.ibestread.com)
>
> 原文:  [Converting a .NET Standard 2.0 library to .NET Core 3.0]( https://andrewlock.net/converting-a-netstandard-2-library-to-netcore-3/ ) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

此文是 [升级至 ASP.NET Core 3.0](/upgrading-to-asp-net-core-3) 第1篇:

1. [转换.NET Standard 2.0类库到.NET Core 3.0](/converting-a-netstandard-2-library-to-netcore-3/)
2. [对比IHostingEnvironment与IHostEnvironment .NET及Core 3.0中的过时类型](/ihostingenvironment-vs-ihost-environment-obsolete-types-in-net-core-3/)
3. [不要在Startup类的构造函数中使用依赖注入](/avoiding-startup-service-injection-in-asp-net-core-3/)
4. [将终端中间件转换为端点路由](/converting-a-terminal-middleware-to-endpoint-routing-in-aspnetcore-3/)
5. [将集成测试升级至.NET Core 3.0](/converting-integration-tests-to-net-core-3/)

这是从ASP.NET Core 2.x升级到ASP.NET Core 3.0的系列的第1篇文章。 我不会讨论诸[Blazor](https://docs.microsoft.com/en-us/aspnet/core/blazor/?view=aspnetcore-3.0)或[gRPC](https://docs.microsoft.com/en-us/aspnet/core/tutorials/grpc/grpc-start)等比较大的新特性。 相反，我将介绍一些令人困惑的事情，例如，如何将类库升级到ASP.NET Core 3.0目标，如何使用基于通用主机的服务器以及如何使用端点路由。

如果打算从ASP.NET Core 2.x升级到3.0，我强烈建议您按照[迁移指南](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30)进行操作，阅读有关[探索ASP.NET Core 3.0](/exploring-asp-net-core-3/)的系列文章，或查看[Rick Strahl的有关转换ASP.NET Core 3.0应用程序](https://weblog.west-wind.com/posts/2019/Sep/24/Upgrading-my-AlbumViewer-Sample-to-NET-Core-30)的文章。 

在这篇文章中，我将描述.NET Standard 2.0类库转换为.NET Core 3.0的步骤，及遇到的一些问题。 在这篇文章中我们将只研究类库项目的转换。

假设您有一个或多个类库，并准备升级至.NET Core 3.0，按照下面不同情况，将类库的依赖关系进行拆分：

- [您的类库，不依赖其他的类库](/converting-a-netstandard-2-library-to-netcore-3/#不依赖其它类库)
- [你的类库，依赖了Microsoft.Extensions.*相关包](/converting-a-netstandard-2-library-to-netcore-3/#依赖了Microsoft-Extensions-相关包)
- [你的类库，依赖了ASP.NET Core相关类库](/converting-a-netstandard-2-library-to-netcore-3/#依赖了ASP-NET-Core相关包)

<!-- more --> 

## 把类库从.NET Standard 2.0升级到.NET Core 3.0是必要的吗

这个不太好回答，因为.NET Core 3.0中的一些改动。[.NET Core 3.0引入一个新概念`FrameworkReference`](/exploring-the-new-project-file-program-and-the-generic-host-in-asp-net-core-3/#新的项目文件及目标框架)，它类似于ASP.NET Core 2.x中的*Microsoft.AspNetCore.App*元包。但是`FrameworkReference`并不是以Nuget包方式引用的。它是跟随.NET Core运行时一起安装。

如果您的类库有引用元包（以前是NuGet元包，但现在已改为*share framework*的一部分预安装的包），那么这种情况就会有问题。 我将组合多种目标框架的方式，让您大致了解如何升级类库，并在.NET Core 3.0中使用。

### 不依赖其它类库

让我们从最简单的情况开始，您的类库不依赖其它类库。

**问：我的类库只依赖.NET Standard 2.0，不依赖其他类库**

从理论上讲，您无需更改此类库。 .NET Core 3.0支持.NET Standard 2.1，并且还支持.NET Standard 2.0。

保持.NET Standard 2.0为目标，您将可以在.NET Core 3.0应用程序中使用它，并且也可以在.NET Core 2.x，.NET Framework 4.6.1以及Xamarin中使用它。

**问：是否应该更新为目标.NET Standard 2.1**

如果目标设置为.NET Standard 2.0，可以允许大量的其它框架使用您的类库。 但升级到.NET Standard 2.1后将极大地限制这一点。 升级后，您将不能使用.NET Core 2.x，.NET Framework，Unity或更早的Mono / Xamarin版本的类库。 所以，不要更改目标至.NET Standard 2.1。

另外说一点，.NET Standard 2.1中包含了一些提升性能的函数或方法，比如`IAsyncEnumerable <>`。如果既想使用这些新函数或方法，又想让更多的人使用您的类库，您可以针对2.0和2.1，使用条件编译来支持多个平台。 如果确定您的类库仅在.NET Standard 2.1的平台上使用，修改`.csproj`文件中的`<TargetFramework>`值为`netstandard2.1`即可。

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.1</TargetFramework>
    <LangVersion>8.0</LangVersion>
  </PropertyGroup>
</Project>
```

> 如果升级到目标.NET Standard 2.1，则最好使用C＃8。 

**问：我的类库目标是.NET Core 2.x，并不依赖其它Nuget包**

这种情况与上一种基本相同。 .NET Core 3.0应用程序可以使用以.NET Core 3.0或更低版本为目标的任何类库，因此除非您有强迫症，否则无需更新类库。 如果您以.NET Core 2.x为目标，则可以使用该平台可用的所有功能（比.NET Standard 2.0中的功能更多）。 如果升级到.NET Core 3.0，则可以再使用到更多功能，但.NET Core 2.x应用程序将无法再使用中的库。

**问：我的类库依赖了其它Nuget包**

当您依赖其他NuGet软件包时，事情会变得有些棘手。

如果你依赖的类库的依赖（向上依赖）不是Microsoft.AspNetCore.\*和Microsoft.Extensions.\*，那就无需担心，只要你依赖的类库支持你的目标框架即可。但是如果你依赖的类库依赖了微软的库，那处理起来就要特别小心。

### 依赖了Microsoft.Extensions.*相关包

Microsoft.Extensions.\*库提供了一些基础功能，例如[依赖项注入](https://www.nuget.org/packages/Microsoft.Extensions.DependencyInjection/)，[配置](https://www.nuget.org/packages/Microsoft.Extensions.Configuration/)，[日志记录](https://www.nuget.org/packages/Microsoft.Extensions.Logging/)和[通用主机](https://www.nuget.org/packages/Microsoft.Extensions.Hosting/)。 这些功能全部由ASP.NET Core应用程序使用，但是您也可以在没有ASP.NET Core的情况下，使用它们来创建各种其他服务或控制台应用程序。

使用Microsoft.Extensions.\*的好处是，使您可以轻松创建使用.NET Core生态的项目，使得用户使用基础功能变得非常简单。

在.NET Core 3.0中，Microsoft.Extensions.\*类库的主要版本均升至3.0.0。 同时目标也支持`netstandard2.0`和`netcoreapp3.0`。鉴于.NET Core 2.x应用程序支持.NET Standard 2.0，那么可以在.NET Core 2.x中使用3.0.0 Microsoft.Extensions.\*类库吗？

如果您使用的是控制台应用程序，升级Microsoft.Extensions.\*到*3.0.0*，是没问题的。并且可以使用*3.0.0*中的那些新的类对象。

但如果你使用的是ASP.NET Core 2.x应用程序呢？

是不行的。尽管您可以添加对3.0.0库的引用，但是在ASP.NET Core 2.x应用程序中，核心库也依赖于Microsoft.Extensions.\*。 当您编译应用程序时，会出现以下错误：

```bash
C:\repos\test\test.csproj : warning NU1608: Detected package version outside of dependency constraint: Microsoft.AspNetCore.App 2.1.1 requires Microsoft.Extensions.Configuration.Abstractions (>= 2.1.1 && < 2.2.0) but version Microsoft.Extensions.Configuration.Abstractions 3.0.0 was 
resolved.
C:\repos\test.csproj : error NU1107: Version conflict detected for Microsoft.Extensions.Primitives. Install/reference Microsoft.Extensions.Primitives 3.0.0 directly to project PwnedPasswords.Sample to resolve this issue.
```

你只能接受ASP.NET Core 2.x应用程序不能使用Microsoft.Extensions.\*的*3.0.0*的事实了。

现在，我们来看看Microsoft.Extensions.\*库对您自己类库的影响。

**问：我的类库使用了Microsoft.Extensions.\*，并只在.NET Core 3.0下使用**

如果您的类库只是对内使用的（无需考虑其它外部类库引用您），则可以仅指定.NET Core 3.0目标。 在这种情况下，将3.0.0作为目标是有意义的。

**问：我的类库使用了Microsoft.Extensions.\*，并在.NET Core 2.x和3.0下都有使用**

在大多数情况下，Microsoft.Extensions.\*的2.x和3.0版本之间几乎没有什么区别。 特别是Microsoft.Extensions.\*.Abstractions的相关库：*Microsoft.Extensions.Configuration.Abstractions* ...

例如，对于*Microsoft.Extensions.Configuration.Abstractions*，在版本2.2.0和3.0.0之间，实际上添加了一个API：

![](https://cdn.ibestread.com/img/libs_fuget.png)

> 这是在[https://fuget.org](https://fuget.org/)上使用[查看API差异](https://www.fuget.org/packages/Microsoft.Extensions.Configuration.Abstractions/3.0.0/lib/netstandard2.0/diff/2.2.0/)功能的截图

这意味着您的类库可以继续使用2.x版本为目标。 在ASP.NET Core 2.x应用程序中使用时，将像以前一样使用**2.x.x**库。 但是，当您在ASP.NET Core 3.0中引用库时，由于NuGet程序包解析规则，库的**2.x**依赖项将自动升级到**3.0.0**版本。

通常，您要避免自动升级，因为主要版本的升级意味着破坏更改。 您不能保证前一个版本目标框架上的代码，在下一个主要版本目标框架上还能正常运行。

现在，我们已经确定了3.0.0版本与上个版本大致相同的，因此无需担心！ 为了进一步证明这样做是可以的，这是[Serilog的Microsoft.Extensions.Logging集成包使用的方法]((https://github.com/serilog/serilog-extensions-logging))。 该软件包保留目标.NET Standard 2.0并引用Microsoft.Extensions.Logging的2.0.0版本，实际上可以在ASP.NET Core 3.0应用程序中使用：

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>netstandard2.0</TargetFrameworks>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Serilog" Version="2.8.0" />
    <PackageReference Include="Microsoft.Extensions.Logging" Version="2.0.0" />
  </ItemGroup>

</Project>
```

> 需要注意.NET Framework目标，您需要为Microsoft.Extensions.\*库使用重定向绑定。 

**问：我的类库使用了Microsoft.Extensions.\*，并且分别在.NET Core 2.x和3.0使用了不同的版本**

并非所有的类库都可以采用静默方式安全地升级。 例如，[*Microsoft.Extensions.Options*](https://www.fuget.org/packages/Microsoft.Extensions.Options/3.0.0/lib/netstandard2.0/diff/2.2.0/)， 在3.0.0中，从[`OptionsWrapper<>`](https://github.com/aspnet/Extensions/blob/20dbe660793d5280aead975cd5e6e8d6b280ea3c/src/Options/Options/src/OptionsWrapper.cs)中删除了`Add`，`Get`和`Remove`方法。 如果您的类库使用了这些方法，那么使用在ASP.NET Core 3.0中运行的应用程序，将在运行时抛出`MethodNotFoundException`异常。

上面的示例是假象出来的（您不太可能在库中使用`OptionsWrapper<>`），但是如果使用[`IdentityModel`](https://www.fuget.org/packages/IdentityModel/4.0.0/lib/netstandard2.0/diff/3.10.10/)时，就会经常遇到此问题。 您必须非常小心地处理在所有依赖项中引用该库的主版本相同，否则可能会在运行时抛出`MethodNotFoundExceptions`异常。

>升级到.NET Core 3.0后，使用`IdentityModel`可能会遇到的问题是`CryptoRandom.CreateUniqueId()`方法。 从[fuget.org](https://www.fuget.org/packages/IdentityModel/4.0.0/lib/netstandard2.0/diff/3.10.10/)比较中可以看到，在4.0.0版本中该方法的默认参数已更改。`IdentityModel`升级采用的是，[使用运行时的破坏性变更，替代编译时的破坏性变更](https://haacked.com/archive/2010/08/10/versioning-issues-with-optional-arguments.aspx/)。

![](https://cdn.ibestread.com/img/libs_identitymodel.png)

那我们应该如何处理这种情况呢？目前最好的方式是分别针对不同的目标*.NET Standard 2.0*和*.NET Core 3.0*，使用MSBuild的不同条件指定正确版本的类库。

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>netstandard2.0;netcoreapp3.0</TargetFrameworks>
  </PropertyGroup>

  <ItemGroup Condition="'$(TargetFramework)' == 'netcoreapp3.0'">
    <PackageReference Include="Microsoft.Extensions.Options" Version="3.0.0" />
    <PackageReference Include="IdentityModel" Version="4.0.0" />
  </ItemGroup>

  <ItemGroup Condition="'$(TargetFramework)' != 'netcoreapp3.0'">
    <PackageReference Include="Microsoft.Extensions.Options" Version="2.2.0" />
    <PackageReference Include="IdentityModel" Version="3.10.10" />
  </ItemGroup>

</Project>
```

在上面的示例中，我展示了一个类库依赖于*Microsoft.Extensions.Options*和*IdentityModel*的情况。 从技术上讲，这两个软件包的最新版本都支持.NET Standard 2.0，但正如我所讨论的，这些差异还是细微的差别。

- 当ASP.NET Core 2.x应用程序依赖于上述库时，它将使用**2.2.0**版本的*Microsoft.Extensions.Options*库和**3.10.10**版本的*IdentityModel*。 
- 当ASP.NET Core 3.0应用程序依赖于上述库时，它将使用**3.0.0**版本的*Microsoft.Extensions.Options*库和**4.0.0**版本的*IdentityModel*。 

这种方式的主要缺点：项目文件较为复杂，代码中也到处充斥着`#if`，`#elseif` 等编译条件，另外还需要增加额外的测试。但是这种方式是最安全的。

我还剩下一种情况没有说：.NET Core 2.x应用程序（非ASP.NET Core），并且使用了3.0.0版本的Microsoft.Extensions.\*库（或4.0.0版本的*IdentityModel*），并且正在使用通过上述方法构建。 在这种情况下，运行时会使用`netstandard2.0`，您可能又回到`MethodNotFound`的老路上。 幸运地是，这似乎比较小众，通常不予支持。

### 依赖了ASP.NET Core相关包

你的类库依赖于ASP.NET Core的相关库。 包括所有以Microsoft.Extensions.\*开头的类库（[有关完整列表，请参阅迁移指南](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30?view=aspnetcore-3.0&tabs=visual-studio#remove-obsolete-package-references)）。 这些NuGet软件包不再推送到[https://nuget.org](https://nuget.org)，因此您无法通过Nuget引用它们！它们作为ASP.NET Core 3.0共享框架的一部分被安装。您可以使用`<FrameworkReference>`方式一次性引用，而无需每个单独引用，ASP.NET Core 3.0中的所有API均可用。

`<FrameworkReference>`有个很不错的功能，它不需要生成额外的类库到输出文件夹中。*MSBuild*知道你应用程序会调用哪些Api。因此你的[应用程序发布体积得到了缩减](https://stackoverflow.com/a/57760357/6375486)。

> 并非*Microsoft.AspNetCore.App*元包中的所有类库都已移到了ASP.NET Core 3.0共享框架。 除<FrameworkReference>之外，[迁移文档中列出的软件包](https://docs.microsoft.com/en-us/aspnet/core/migration/22-to-30?view=aspnetcore-3.0&tabs=visual-studio#add-package-references-for-removed-assemblies)，仍需要使用Nuget引用。 例如：[EF Core](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore) ， [JSON.NET MVC](https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.NewtonsoftJson)和 [Identity UI](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?view=aspnetcore-3.0) 之类的这些包。

**问：你的类库仅需要ASP.NET Core  3.0目标**

如此[StackOverflow问题](https://stackoverflow.com/questions/57760356/how-do-i-use-asp-net-core-3-0-types-from-a-library-project-for-shared-controller)中所述，这种情况处理最为简单：直接升级从目标框架2.x升级到3.0。

删除已过时的包，并改为`<FrameworkReference>`：

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netcoreapp3.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>

</Project>
```

所有ASP.NET Core API都是可以使用智能提示，而您不必担心试图在各个软件包中寻找所需的API。

如果您还需要支持.NET Core 2.x，那么事情会变得更加复杂。

**问：我的类库需要同时支持 ASP.NET Core 2.x and ASP.NET Core 3.0** 

处理这种情况的唯一的方法是：使用我们先前处理Microsoft.Extensions.\*和*IdentityModel*的方式。 同时支持*.NET Standard 2.0*目标框架（以支持.NET Core 2.x和.NET Framework 4.6.1+），和*.NET Core 3.0*目标框架。 并使用条件分别引用*ASP.NET Core 2.x*和*ASP.NET Core 3.0*下的程序包，参考项目文件：

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>netstandard2.0;netcoreapp3.0</TargetFrameworks>
  </PropertyGroup>

  <ItemGroup Condition="'$(TargetFramework)' == 'netcoreapp3.0'">
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>

  <ItemGroup Condition=" '$(TargetFramework)' != 'netcoreapp3.0'">
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Cors" Version="2.1.3" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Formatters.Json" Version="2.1.3" />
    <PackageReference Include="Microsoft.Extensions.Configuration" Version="2.1.1" />
  </ItemGroup>

</Project>
```

上面几乎涵盖了您可以遇到的所有情况。 支持这些较旧版本类库会非常复杂，因此，是否值得还取决于您。 但是，由于ASP.NET Core 2.1是.NET Core的LTS版本（并在.NET Framework上受“永久”支持），我相信许多人会在这个版本上停留一段时间。

> 除了把目标框架设置为.NET Standard 2.0外，您还可以像[Damian Edwards TagHelperPack](https://github.com/DamianEdwards/TagHelperPack/blob/a1c0a7e964da9725052bb59984f6fef3db794e91/src/TagHelperPack/TagHelperPack.csproj)中那样明确定位使用.NET Core 2.1和.NET Framework 4.6.1。 最终结果是一样的。

## 总结

在这篇文章中，我尝试根据其依赖关系，分解所有不同的方法，来升级类库以支持.NET Core 3.0。 

- 如果您不依赖任何包，又或者没有依赖ASP.NET Core/Microsoft.Extensions.\*，那么升级就不会有任何问题。 
- 如果您对Microsoft.Extensions.\*有依赖，则可以在不升级包的情况下解决问题，但是，您可能需要在*.csproj*文件中，添加不同目标框架的条件引用。 
- 如果您对*ASP.NET Core*有依赖，并且还需要支持2.x和3.0，那么可以肯定的是，需要将MSBuild条件添加到*.csproj*文件中。