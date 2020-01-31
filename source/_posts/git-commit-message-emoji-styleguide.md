---
title: Git提交时的emoji图标使用指南
tags: 
  - git commit
  - emoji
  - 指南

categories:
  - 随笔

date: 2020-01-31 18:20:00 
---

在执行`git commit`时使用emoji为本次提交打上一个表情标签，使得此次*commit*的工作更表意的显示，方面在整体历史提交记录中易于区分和查找。如下图：

![示例提交emoji](https://cdn.ibestread.com/img/styleguide-git-commit-message.png)

<!-- more --> 

## 提交格式

在执行`git commit`时，遵循以下格式：

```ruby
:emoji1: :emoji2: 主题

提交信息主体

Ref <###>
```

初始化仓库时的示例：

```bash
git commit -m ":tada: Initialize Repo"
```

##  建议使用的Emoji

| Emoji | Emoji 编码 | 描述 |
|:---:|:---:|---|
| :art: | `:art:` | 优化代码或调整结构 |
| :newspaper: | `:newspaper:` | 添加**新文件** |
| :pencil: | `:pencil:` | 较小的修改或Bug修复 |
| :racehorse: | `:racehorse:` | 提升性能 |
| :books: | `:books:` | 编写文档 |
| :bug: | `:bug:` | 报告Bug 配合@Akinix使用 |
| :ambulance: | `:ambulance:` | 修复Bug |
| :penguin: | `:penguin:` | 修复 **Linux** Bug |
| :apple: | `:apple:` | 修复 **Mac OS** Bug |
| :checkered_flag: | `:checkered_flag:` | 修复 **Windows** Bug |
| :fire: | `:fire:` | 删除文件 |
| :tractor: | `:tractor:` | 调整文件结构，通常与 :art: 一起使用 |
| :hammer: | `:hammer:` | 开始重构代码 |
| :umbrella: | `:umbrella:` | 添加 **测试** |
| :microscope: | `:microscope:` | 添加 **代码覆盖率** |
| :green_heart: | `:green_heart:` | 修复**CI**相关Bug |
| :lock: | `:lock:` | **安全** 相关 |
| :arrow_up: | `:arrow_up:` | 升级**依赖** |
| :arrow_down: | `:arrow_down:` | 降级**依赖** |
| :fast_forward: | `:fast_forward:` | 向前合并 |
| :rewind: | `:rewind:` | 后退 |
| :shirt: | `:shirt:` | 移除代码检查中的警告 |
| :lipstick: | `:lipstick:` | 修改 **UI** |
| :wheelchair: | `:wheelchair:` | 改进 **辅助功能** |
| :globe_with_meridians: | `:globe_with_meridians:` | 本地化 |
| :construction: | `:construction:` | **WIP**(Work In Progress) ，通常与 `@REVIEW` 一起使用 |
| :gem: | `:gem:` | 新 **Release** |
| :egg: | `:egg:` | 新 **Release** 使用 Python egg |
| :ferris_wheel: | `:ferris_wheel:` | 新 **Release** 使用 Python wheel package |
| :bookmark: | `:bookmark:` | 新 **Tags** |
| :tada: | `:tada:` | 初始提交 |
| :speaker: | `:speaker:` | 添加日志相关 |
| :mute: | `:mute:` | 移除日志相关 |
| :sparkles: | `:sparkles:` | 新增功能 |
| :zap: | `:zap:` | 破坏性变更，通常与 `@CHANGED` 一起使用 |
| :bulb: | `:bulb:` | 新想法，通过与 `@IDEA` 一起使用 |
| :snowflake: | `:snowflake:` | 修改 **配置**，通常与 :penguin: ，:ribbon: ，:rocket: 一起使用 |
| :ribbon: | `:ribbon:` | 自定义，通常与 `@HACK` 一起使用 |
| :rocket: | `:rocket:` | **DevOps** 相关 |
| :elephant: | `:elephant:` | **PostgreSQL** 数据库相关 |
| :dolphin: | `:dolphin:` | **MySQL** 数据库相关 |
| :leaves: | `:leaves:` | **MongoDB** 数据库相关 |
| :bank: | `:bank:` | **其他数据库** 相关 |
| :whale: | `:whale:` | **Docker** 配置 |
| :handshake: | `:handshake:` | 合并文件 |
| :cherries: | `:cherries:` | [**Cherry-Pick**](https://git-scm.com/docs/git-cherry-pick) |

## 参考

[Atom Git提交格式](https://github.com/atom/atom/blob/master/CONTRIBUTING.md#git-commit-messages)

[风格指南](https://github.com/slashsbin/styleguide-git-commit-message)

[更多emoji图标提交用法](https://gitmoji.carloscuesta.me/)