---
title: Windows下的软件管理利器-Chocolatey
tags: 
  - 效率
  - Chocolatey
  - Windows
  - 软件

categories:
  - 随笔

layout: slides
slide:
  theme: night

date: 2020-02-03 19:37:00 
---

## Chocolatey <small>Windows下的软件管理利器</small>
<!-- .slide: data-background="#49B1F5" -->

[Chocolatey](https://chocolatey.org/)是Windows平台上的软件管理器，通过它可以集中安装、管理、更新各种各样的软件。

类似 Linux 下的apt-get 、yum等程序包管理器。 

===

## 安装Chocolatey
<!-- .slide: data-transition="concave" data-background="#C7916B" -->

### 以管理员的方式打开 PowerShell

 `右击` 左下角 `Windows 图标`，选择 `Windows PowerShell(管理员)(A)` 

===

### 输入安装命令安装 Chocolatey
<!-- .slide: data-transition="fade" data-background="#00C4B6" -->

```bash
Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```

===

### 检查是否安装成功

在 PowerShell 终端输入 `choco` 或 `choco -?` 看到下面内容即表示已安装成功： 
<!-- .slide: data-transition="convex" data-background="#1B9EF3" -->

```bash
This is a listing of all of the different things you can pass to choco.

Commands

 * list - lists remote or local packages
 * find - searches remote or local packages (alias for search)
 * search - searches remote or local packages (alias for list)
 * info - retrieves package information. Shorthand for choco search pkgname --exact --verbose
 * install - installs packages from various sources
 ...
```

===

## Choco 安装软件
<!-- .slide: data-transition="zoom" data-background="#F47466" -->

### 在终端中查找软件包

在终端中输入 `choco` 搜索命令：

```bash
choco search nodejs # 这是搜索命令
# 下面是搜索结果：
Chocolatey v0.10.15
nodejs 13.7.0 [Approved]
nodejs.commandline 6.11.0 [Approved]
nodejs.install 13.7.0 [Approved]
nodejs-lts 12.14.1 [Approved]
nvs 1.5.2 [Approved] Downloads cached for licensed users
...
```

===

###  在网页端查找软件包
<!-- .slide: data-transition="convex" data-background="#69C282" -->

-  登录 `choco` 软件包网站：[chocolatey.org](https://chocolatey.org/packages) 

- 在搜索框搜索要安装的软件

	![](https://cdn.ibestread.com/img/choco-6.png)

===

## 安装软件包
<!-- .slide: data-background="#49B1F5" -->

在 PowerShell 终端输入安装命令

```bash
choco install nodejs -y
```

常用软件：

```bash
choco install python -y # 安装python
choco install git.install -y # 安装git
choco install notepadplusplus.install -y # 安装notepad++
choco install googlechrome -y # 安装chrome
choco install typora -y # 安装typora
choco install sourcetree -y # 安装Source Tree
```


