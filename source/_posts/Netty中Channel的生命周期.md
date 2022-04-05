---
title: Netty中Channel的生命周期
tags:
  - Netty
categories: 正常的文章
date: 2022-03-06 22:57:33
---

# Netty中Channel的生命周期

## 服务端

### A `Channel` in an parent `EventLoopGroup`

Registered -> Bind -> Active -> [Read -> Read Complete] -> Close -> Inactive -> Unregistered


### A `Channel` in an child `EventLoopGroup`

Registered -> Active -> [(Read/Write -> Read Complete/Flush)、Event Triggered、Exception Caught] -> Inactive ->Unregistered


## 客户端

Registered -> Connect -> Active(Lazy Active) -> [(Read/Write -> Read Complete/Flush)、Event Triggered、Exception Caught、Disconnect] -> Close -> Inactive -> Unregistered

