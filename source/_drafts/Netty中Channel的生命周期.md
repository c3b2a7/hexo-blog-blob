---
title: Channel's Life Cycle In Netty
tags: [Netty]
---

# Channel's Life Cycle In Netty

## Server Side

### A `Channel` in an parent `EventLoopGroup`

Registered -> Bind -> Active -> [Read -> Read Complete] -> Close -> Inactive -> Unregistered



### A `Channel` in an child `EventLoopGroup`

Registered -> Active -> [(Read/Write -> Read Complete/Flush)、Event Triggered、Exception Caught] -> Inactive ->Unregistered



## Client Side

Registered -> Connect -> Active(Lazy Active) -> [(Read/Write -> Read Complete/Flush)、Event Triggered、Exception Caught、Disconnect] -> Close -> Inactive -> Unregistered

