---
title: swift-nio源码阅读系列1
tags: [swift,backend]
date: 2018-05-11 22:33:34
desc:
---

<!--more-->
## 概要
### swift-nio
*Apple* 今年三月份的时候开源了[swift-nio][1]这个库:
> SwiftNIO is a cross-platform asynchronous event-driven network application framework for rapid development of maintainable high performance protocol servers & clients.   
It's like Netty, but written for Swift.

`Netty`是`java`端一款开源的网络层框架。`swift-nio`就是`swift`版的`netty`，上面那句官方介绍已经把`swift-nio`说的很清楚了，**跨平台**,**异步**,**网络应用框架**,服务于高性能的**服务器或客户端协议**，最后一点什么意思呢？`swift-nio`是一个通用的网络层应用，它只提供

[1]: https://github.com/apple/swift-nio  "swift-nio"