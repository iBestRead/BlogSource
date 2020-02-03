---
title: 如何使用Cake.CoreCLR或Cake全局工具，在Linux系统上使用Cake构建系统
tags: 
  - .NET CORE
  - CAKE
  - DEVOPS
  - DOCKER

date: 2020-02-03 15:38:00
---

> 译者:  [Akini Xu](/)
>
> 原文:  [How to build with Cake on Linux using Cake.CoreCLR or the Cake global tool]( https://andrewlock.net/how-to-build-with-cake-on-linux-using-cake-coreclr-or-the-cake-global-tool/ ) 
>
> 作者:  [Andrew Lock](https://andrewlock.net/about/)
>

在本文中，我会演示使用[Cake的构建系统](https://cakebuild.net/)在Linux上构建.NET Core项目的两种方式：一是使用`Cake.CoreCLR`库，二是使用`Cake.Tool .NET Core`全局工具。 这篇文章仅涉及使用`Cake`，并不详细说明如何编写自定义的Cake构建脚本。 我建议阅读[Muhammad Rehan Saeed](https://rehansaeed.com/cross-platform-devops-net-core/)的文章，其中提供了Cake构建脚本，或者我之前的文章[有关在Docker中使用Cake]( https://andrewlock.net/building-asp-net-core-apps-using-cake-in-docker/ )。

<!-- more -->

## Cake，Cake.CoreCLR，Cake.Tool

在[之前的文章中](https://andrewlock.net/building-asp-net-core-apps-using-cake-in-docker/)，我描述了如何使用Mono在Linux上运行.NET Framework的Cake。那个时候，我们可以这种方式来编译.NET Framework的库，并运行单元测试。

目前，如果只是编译（不需要运行）的话，可以引用 [*Microsoft.NetFramework.ReferenceAssemblies*](https://andrewlock.net/using-reference-assemblies-to-build-net-framework-libararies-on-linux-without-mono/) 包在Linux上编译。无需安装Mono，因此，我认为Cake的.NET Framework版本，不再是在Linux上编译的最佳选择。

[现在有3个不同版本的Cake](https://github.com/cake-build/cake/issues/1751#issuecomment-487494424)：

- [Cake.Tool](https://www.nuget.org/packages/Cake.Tool/) ：面向.NET Core 2.1的全局工具。
- [Cake.CoreCLR](https://www.nuget.org/packages/Cake.CoreCLR/) ：面向.NET Core 2.0的命令行程序。
- [Cake](https://www.nuget.org/packages/Cake/) ：面向.NET Framework 4.61/Mono 的命令行程序。

具体哪种选择哪个版本，可能取决于您程序的实际环境。 这些工具的优点是，可以在不同版本之间进行切换，而不必完全更改实际的*build.cake*构建脚本。主要区别在于如何获取和运行Cake的引导脚本。

>请注意，在本文中，假设您已经安装了正确版本的.NET Core。 有关在引导脚本中安装.NET Core SDK的示例，请从Cake项目中[查看此示例](https://github.com/cake-build/cake/blob/06e7983704/build.sh)。

## 通过Cake.CoreCLR在Linux上构建

可以通过使用`Cake.CoreCLR`来转换基于Mono的Cake构建。这种方式相对比较容易，虽然花了一些时间来查找引导脚本。下面2种方式都可以获取到正确引导脚本：

- 使用.NET Core CLI命令还原`Cake.CoreCLR`包。
- 使用`curl`下载`Cake.CoreCLR`包，然后手动解压。

> 注意： `dotnet CLI`允许还原已添加到项目中的NuGet包，但是**不允许**您还原**随意**（这个词在这有点难懂）软件包。 要解决此问题，您可以执行以下操作：

```bash
dotnet new classlib -o "$TEMP_DIR" --no-restore
dotnet add "$TEMP_PROJECT" package Cake.CoreCLR --package-directory "$TOOLS_DIR" --version "$CAKE_VERSION"
rm -rf "$TEMP_DIR"
```

上面的脚本，完成了4件事：

- 使用`dotnet new`在`$TEMP_DIR`目录内，创建了一个临时的.NET Core项目。
- 添加`Cake.CoreCLR`引用到这个临时项目。
- 还原这个项目的引用到指定目录` $TOOLS_DIR `。
- 删除临时项目。

运行完上面的脚本后，`Cake.CoreCLR`会被下载并解压到tools目录。

下面的脚本，大致上完成了引导功能，在本文的最后会列举其他脚本示例：

```bash
#!/usr/bin/env bash

# Define directories.
SCRIPT_DIR=$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )
TOOLS_DIR=$SCRIPT_DIR/tools
TEMP_DIR=$TOOLS_DIR/build
TEMP_PROJECT=$TEMP_DIR/build.csproj

# Define default arguments.
SCRIPT="build.cake"
CAKE_VERSION="0.33.0"
CAKE_ARGUMENTS=()

# Parse arguments.
for i in "$@"; do
    case $1 in
        -s|--script) SCRIPT="$2"; shift ;;
        --cake-version) CAKE_VERSION="$2"; shift ;;
        --) shift; CAKE_ARGUMENTS+=("$@"); break ;;
        *) CAKE_ARGUMENTS+=("$1") ;;
    esac
    shift
done

CAKE_PATH="$TOOLS_DIR/cake.coreclr/$CAKE_VERSION/Cake.dll"

if [ ! -f "$CAKE_PATH" ]; then
    echo "Restoring Cake..."

    # Make sure the tools folder exists
    if [ ! -d "$TOOLS_DIR" ]; then
        mkdir "$TOOLS_DIR"
    fi

    # Build the temp project and restore Cake
    dotnet new classlib -o "$TEMP_DIR" --no-restore
    dotnet add "$TEMP_PROJECT" package Cake.CoreCLR --package-directory "$TOOLS_DIR" --version "$CAKE_VERSION"

    rm -rf "$TEMP_DIR"
fi

# Start Cake
exec dotnet "$CAKE_PATH" "$SCRIPT" "${CAKE_ARGUMENTS[@]}"
```

该脚本的前半部分是解析命令行参数并定义默认值。 再就是检查`Cake.dll`文件是否存在，如果不存在，就使用刚才的逻辑来还原它。 最后，使用`dotnet Cake.dll`执行`Cake`。

>  请注意，我固定Cake.CoreCLR版本在0.33.0，以确保我们获取的Cake的版本与编译服务器一致。 

当你运行该脚本，你会看到类似的输出，它显示构建过程是如何工作的： 

```bash
Restoring Cake...
Getting ready...
The template "Class library" was created successfully.
  Writing /tmp/tmp8yvA9D.tmp
info : Adding PackageReference for package 'Cake.CoreCLR' into project '/sln/tools/build/build.csproj'.
info : Restoring packages for /sln/tools/build/build.csproj...
info :   GET https://api.nuget.org/v3-flatcontainer/cake.coreclr/index.json
info :   OK https://api.nuget.org/v3-flatcontainer/cake.coreclr/index.json 438ms
info :   GET https://api.nuget.org/v3-flatcontainer/cake.coreclr/0.33.0/cake.coreclr.0.33.0.nupkg
info :   OK https://api.nuget.org/v3-flatcontainer/cake.coreclr/0.33.0/cake.coreclr.0.33.0.nupkg 25ms
info : Installing Cake.CoreCLR 0.33.0.
info : Package 'Cake.CoreCLR' is compatible with all the specified frameworks in project '/sln/tools/build/build.csproj'.
info : PackageReference for package 'Cake.CoreCLR' version '0.33.0' added to file '/sln/tools/build/build.csproj'.
info : Committing restore...
info : Generating MSBuild file /sln/tools/build/obj/build.csproj.nuget.g.props.
info : Generating MSBuild file /sln/tools/build/obj/build.csproj.nuget.g.targets.
info : Writing assets file to disk. Path: /sln/tools/build/obj/project.assets.json
log  : Restore completed in 2.95 sec for /sln/tools/build/build.csproj.
```

通过NuGet包还原是一种很聪明的方法，但是有点过于复杂。 另一个选择是直接下载NuGet并将其解压。 您可以使用下面脚本替换原脚本中的还原包的代码：

```bash
curl -Lsfo "$TOOLS_DIR/cake.coreclr.zip" "https://www.nuget.org/api/v2/package/Cake.CoreCLR/$CAKE_VERSION" \
    && unzip -q "$TOOLS_DIR/cake.coreclr.zip" -d "$TOOLS_DIR/cake.coreclr/$CAKE_VERSION" \
    && rm -f "$TOOLS_DIR/cake.coreclr.zip"

if [ $? -ne 0 ]; then
    echo "An error occured while installing Cake."
    exit 1
fi
```

这直接下载的NuGet的.zip文件，再解压，最后删除该压缩文件。 

>  请注意，您的环境必须有unzip命令。（一般不会默认安装的） 

无论使用那种方法，都能够下载`Cake.CoreCLR `NuGet包。另一种方法是安装.NET Core 全局工具。 

## 使用Cake全局工具在Linux上构建

.NET Core 2.1引入了[全局工具](https://docs.microsoft.com/en-us/dotnet/core/tools/global-tools)的概念。 这些CLI工具可以全局安装在您的计算机上，实际上它只是控制台应用程序。 在[上一篇文章中](https://andrewlock.net/creating-a-net-core-global-cli-tool-for-squashing-images-with-the-tinypng-api/)，我描述了如何创建.NET Core全局工具。 Cake也提供了可与.NET Core 2.1+一起使用的全局工具。

根据您的使用情况，您可能不想全局安装Cake工具。 也可以通过传递`--tool-path`参数，将全局工具安装到指定文件夹，在这种情况下，您可以安装该工具的多个实例。

> 我的选择是：在每个解决方案内，安装的全局工具，以消除对外界的工具的依赖。这样可以允许每个项目自包含不同版本的工具。唯一不足的就是，会消耗更多的磁盘空间。 

把Cake全局工具安装到本地*tools*文件夹中，可以运行：

```bash
dotnet tool install Cake.Tool --tool-path ./tools --version 0.33.0
```

调用是，运行：

```bash
exec ./tools/dotnet-cake
```

> **注意**，我们使用`--tool-path`安装了该工具，因此运行的时候，我们加上`dotnet-cake`的路径。 如果是全局安装的，则可以从任何文件夹使用`dotnet-cake`（或`dotnet cake`）来运行。

添加下面的引导脚本：

```bash
#!/usr/bin/env bash

# Define directories.
SCRIPT_DIR=$( cd "$( dirname "${BASH_SOURCE[0]}" )" && pwd )
TOOLS_DIR=$SCRIPT_DIR/tools

# Define default arguments.
SCRIPT="build.cake"
CAKE_VERSION="0.33.0"
CAKE_ARGUMENTS=()

# Parse arguments.
for i in "$@"; do
    case $1 in
        -s|--script) SCRIPT="$2"; shift ;;
        --cake-version) CAKE_VERSION="--version=$2"; shift ;;
        --) shift; CAKE_ARGUMENTS+=("$@"); break ;;
        *) CAKE_ARGUMENTS+=("$1") ;;
    esac
    shift
done

# Make sure the tools folder exists
if [ ! -d "$TOOLS_DIR" ]; then
    mkdir "$TOOLS_DIR"
fi

CAKE_PATH="$TOOLS_DIR/dotnet-cake"
CAKE_INSTALLED_VERSION=$($CAKE_PATH --version 2>&1)

if [ "$CAKE_VERSION" != "$CAKE_INSTALLED_VERSION" ]; then
    if [ -f "$CAKE_PATH" ]; then
        dotnet tool uninstall Cake.Tool --tool-path "$TOOLS_DIR" 
    fi

    echo "Installing Cake $CAKE_VERSION..."
    dotnet tool install Cake.Tool --tool-path "$TOOLS_DIR" --version $CAKE_VERSION

    if [ $? -ne 0 ]; then
        echo "An error occured while installing Cake."
        exit 1
    fi
fi


# Start Cake
exec "$CAKE_PATH" "$SCRIPT" "${CAKE_ARGUMENTS[@]}"
```

和之前的引导脚本一样，脚本的前半部分是解析命令行参数并设置默认值。 然后检查是否安装了正确版本的Cake全局工具。 如果版本错误，我们将首先卸载错误版本，再安装正确版本。 最后，我们使用全局工具的路径执行脚本。

> 如果是在Docker中执行此脚本，通常镜像中不会安装Cake工具的任何版本，因此您可以根据需要，移除版本检查那部分的代码。

首次运行脚本时，您会看到已安装了全局工具：

```bash
Installing Cake 0.33.0...
You can invoke the tool using the following command: dotnet-cake
Tool 'cake.tool' (version '0.33.0') was successfully installed.
```

随后，直接执行该工具就好了。

我个人认为，在所有情况下（.NET Core 2.1+），都可以使用全局工具。 全局工具将在[.NET Core 3.0中进行更新](https://www.dotnetcurry.com/dotnet/1495/dotnet-core-global-tools)，但是我相信仍然是在Linux / Mac上使用`Cake`的标准方法。

## 总结

在本文中，我展示了在Linux上运行Cake的两种方法：使用Cake.CoreCLR或.NET Core全局工具Cake.Tool。 一般来说，我建议使用全局工具，因为这是首选方法-[Cake项目本身也是使用.NET Core全局工具构建的](https://github.com/cake-build/cake/blob/06e7983704/build.sh)！

## 参考链接

- [Cake and .NET Core: we must go deeper](https://medium.com/@dmitriy.litichevskiy/cake-and-net-core-we-must-go-deeper-e51102dfa787) by [Dmitriy Litichevskiy](https://medium.com/@dmitriy.litichevskiy)
- [Cross-Platform DevOps for .NET Core](https://rehansaeed.com/cross-platform-devops-net-core/)
- [Building ASP.NET Core apps using Cake in Docker](https://andrewlock.net/building-asp-net-core-apps-using-cake-in-docker/)
- GitHub issue: [Cake on .Net core](https://github.com/cake-build/cake/issues/1751)
- GitHub issue: [Add Cake CoreCLR bootstrapper](https://github.com/cake-build/resources/issues/28)
- [Example bootstrapper script using Cake.Tool (to build the Cake project)](https://github.com/cake-build/cake/blob/06e7983704/build.sh)
- [Example bootstrapper script using Cake.CoreClr](https://github.com/devlead/BitbucketPipelinesShield/blob/master/build.sh) by [Mattias Karlsson](https://github.com/devlead) (Cake maintainer)