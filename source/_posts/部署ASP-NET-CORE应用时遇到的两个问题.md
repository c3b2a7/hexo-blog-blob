---
title: 部署Asp.Net Core应用时遇到的两个问题
tags:
  - Asp.Net Core
  - Nginx
categories: 正常的文章
date: 2020-06-12 20:08:19
---

在一次部署Asp.Net Core应用时遇到了这么几个问题：

1. 启动应用后，从`IConfiguration`中获取不到连接字符串。
2. 使用nginx反代后，`Identity`框架页面跳转后域名被改写成了`localhost`或者`主机名`。

**问题一：**

应用使用sqlite数据库，在程序启动后创建并初始化数据库，本地开发启动并没有报错，但是部署到服务器却获取不到连接字符串，报错：

![](https://raw-1257226137.cos.ap-guangzhou.myqcloud.com/images/20200612191218.png)

百度和谷歌都无果，最初我还以为是程序中某个地方可能出错了，但是在WSL上经过多次测试，发现如果没有在执行文件的目录下启动了项目才会报这样的错误，这个问题很奇怪，猜测是`Directory.GetCurrentDirectory()`获取路径导致，并未求证。知道了问题所在那么解决的办法就很简单了，`cd`到执行文件目录启动即可。

**问题二：**

这个问题是宝塔面板导致的，在设置反代的发送域名后，并没有反应到配置文件中，也就是说在面板上设置`proxy_set_header Host $host;`后文件中并没有改过来，依旧是`localhost`或者`$hostname`，也就导致了页面跳转时域名变成了`localhost`或者`主机名`。解决的办法很简单，只需要手动修改nginx配置文件中的反代发送域名即可。

在这之前，其实还遇到了一个和nginx反代相关的问题：如果使用nginx来代理目录，从而实现一个域名下部署多个项目，这样做之后就导致Asp.net Core应用全部404，网上搜索一番，有些类似的问题解答提到使用`Microsoft.AspNetCore.HttpOverrides`包，但是我照着操作一番后依旧没用，目前比较笨的解决办法就是mvc的路由前手动加前缀，更好的办法目前没有找到，如果你知道，还请评论告知~