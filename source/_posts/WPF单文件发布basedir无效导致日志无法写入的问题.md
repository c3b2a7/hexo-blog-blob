---
title: WPF单文件发布${basedir}无效导致日志无法写入的问题
tags:
  - WPF
  - NLog
  - Asp.Net Core
categories: 正常的文章
date: 2020-06-20 12:51:36
---


## 写在前面

还是`NLog`问题，虽然`NLog`上手简单，配置容易，但是在`NetCore3`下还是有一些坑的，虽然这不是`NLog`的问题。因为这个问题百度一圈都无果，所以在这里记录一下，希望可以帮助到百度的同学（虽然这个站点没有提交到百度收录？逃~

## 问题由来

好，那么现在回到主题。前一阵子在用`WPF`来做一个`First`、`Follow`集算法模拟的工具，项目中使用`NLog`记录日志，在`NetCore 3.1`下作为单文件发布。发布后`File`类型的日志`target`没有将日志如期写入到文件中。并且仅在单文件发布并且`fileName`使用相对路径或者`${basedir}`指定基础目录时，才会出现这个问题，其实这个问题的解决方法很简单，只需稍微改一下就可以解决。

## 解决

使用`${basedir:fixtempdir=true}`指定基础目录。

这个问题主要原因其实是`NetCore3`单文件发布时`AppDomain.BaseDirectory`无法正确引用基础路径的问题，并且微软并没有打算在`NetCore3`修复这个问题，或许`NetCore5`会修复这个问题吧，文末会放几个链接，感兴趣可以去看看。

## 写在最后

又水了一篇文章，最近文章质量有点差，上个月还鸽掉了一篇Aop详解的文章（其实一直都在`draft`中），感觉有点罪恶感 :D



---

*参考：*

- [PublishSingleFile excluding appsettings not working as expected](https://github.com/dotnet/aspnetcore/issues/12621)
- [Single file publish: AppContext.BaseDirectory doesn't point to apphost directory](https://github.com/dotnet/runtime/issues/3704)
- [When .net core app published as single file - ${basedir} is wrong](https://github.com/NLog/NLog/issues/3808)
- [Why cannot write log to files when WPF application publishing as a single file](https://stackoverflow.com/questions/62445319/why-cannot-write-log-to-files-when-wpf-application-publishing-as-a-single-file/62456300#62456300)
- [Basedir layout renderer](https://github.com/nlog/nlog/wiki/Basedir-Layout-Renderer)