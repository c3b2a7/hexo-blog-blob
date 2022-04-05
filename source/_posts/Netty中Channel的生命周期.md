---
title: Netty中Channel的生命周期
tags:
  - Netty
categories: 正常的文章
date: 2022-03-06 22:57:33
---


## 服务端

### 父EventLoopGroup中的Channel

Registered -> Bind -> Active -> [Read -> Read Complete] -> Close -> Inactive -> Unregistered


### 子EventLoopGroup中的Channel

Registered -> Active -> [(Read/Write -> Read Complete/Flush)、Event Triggered、Exception Caught] -> Inactive ->Unregistered


## 客户端

Registered -> Connect -> Active(Lazy Active) -> [(Read/Write -> Read Complete/Flush)、Event Triggered、Exception Caught、Disconnect] -> Close -> Inactive -> Unregistered

