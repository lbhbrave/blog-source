---
title: swift-nio源码阅读系列1
tags: [swift,backend]
date: 2018-05-11 22:33:34
desc:
---

<!--more-->
# 概要
### swift-nio
*Apple* 今年三月份的时候开源了[swift-nio][1]这个库:
> SwiftNIO is a cross-platform asynchronous event-driven network application framework for rapid development of maintainable high performance protocol servers & clients.   
It's like Netty, but written for Swift.

`netty`是`java`端一款开源的网络层框架。`swiftNio`就是`Swift`版的`netty`，上面那句官方介绍已经把`swift-nio`说的很清楚了：
#### **跨平台**    
`SwiftNio`目标旨在支持所有支持`Swift`的平台，目前支持`linux`和`macOS`
#### **网络应用框架**  
`SwiftNIO`服务于高性能的**服务器或客户端协议**,也就是说，`SwiftNIO`是一个通用的网络层*应用*，它使用异步IO提供高性能网络处理，服务于上层协议,类比于`node.js`,它更像`node`底层所使用的`libuv`,提供的**异步IO**,而不是`Express`,`socket.io`这种具体的`http`,`websocket`等协议框架。基于`swift-nio`我们可以实现`swift`版的`Express`！
#### **异步**  

这里的*异步IO*本质上使用的是同步非阻塞模型,以下是从网上摘来的几种IO模型的介绍
> 同步阻塞（Blocking IO）：最简单的一种IO模型，用户线程在进行IO操作的时候通常是个系统调用，用户线程会由用户空间进入内核空间，内核空间数据包准备好后会将数据拷贝到用户空间，这个时候线程在用户态继续执行。

> 同步非阻塞（Non-blocking IO）：同步非阻塞IO即在同步阻塞的基础之上将socket设置为NONBLOCK。这样用户线程在发起IO操作之后可以立即返回，但是用户线程需要不断轮询来请求数据。

> IO多路复用（IO Multiplexing）：即Reactor设计模式，多路复用模型从流程上和同步阻塞的区别不大，主要区别在于操作系统为用户提供了同时轮询多个IO句柄来查看是否有IO事件的接口（如select），这从根本上允许用户可以使用单个线程来管理多个IO句柄的问题。

> 异步IO（Asynchronous IO）：即Proactor设计模式。在异步IO模型中，用户不需要去轮询IO事件，然后才进行数据的读取，处理；在异步IO模型中，IO事件就绪的时候，内核会开启一个独立的内核线程去执行执行IO操作，实现真正的异步IO。这个时候用户线程可以直接读取内核线程准备好的数据。  

`linux`和`macOS`下暂时没有稳定的`异步I/O` `API`,处理socket基本都是使用`epoll`这种IO多路复用模型(mac下为kqueue)。所以`SwiftNIO`的同步非阻塞具体是：
1. 通过将socket设置为`NONBLOCK`使得我们代码变成异步的，使用`promise`,`future`等概念来设置回调
2. 通过`epoll`帮我们把可用的socket取回
3. 执行上述的回调，操作准备就绪的socket，读取或写入数据

以上只是针对网络I/O。对于文件I/O,我们依旧是阻塞模型，一个线程处理一个I/O事件，为了能让文件I/O也变成异步(使用起来像异步)的，SwiftNIO使用线程池来处理文件I/O(其中每个线程都是阻塞形式的，多个线程能提升处理速度，后续会介绍线程池的实现)，一些耗时的同步操作也放在线程池中。说到这里，我想拿出`node`底层使用的异步I/O框架`libuv`的架构图:  
<div align=center>
<img src="https://raw.githubusercontent.com/lbhbrave/blogImageHosting/master/img/libuv.png" width="70%" height="70%">
</div>


将图中的几个关键名词对应到`SwiftNIO`中:     

- `uv_io_t`对应`Selector`,封装了不同系统下的IO多路复用模块(不包括图中的`event ports`)，提供统一接口
- `Thread Pool`对应`BlockingIOThreadPool`,线程池的实现
- `File I/O`对应`NonBlockingFileIO`，使用线程池实现的异步文件I/O
 > IOCP是Windows下提供的异步I/O接口，和`epoll`这种有本质上的区别，上面已经提到过，目前SwiftNIO还不支持Windows平台

这张图就可以当作SwiftNIO的异步I/O的架构了。


`netty`是基于`java`中的`NIO`框架实现的，而`java`的`NIO`框架提供了上面描述的同步非阻塞模型下异步能力，所以`SwiftNIO`首先实现了类似于`java`里的`NIO`框架，然后在这之上又实现了`netty`。  
`SwiftNIO`的主要贡献者中就有`netty`的核心开发者，此外，`SwiftNIO`中的核心概念和思想与`netty`如出一辙,包括命名，所以这个`Swift`版的`netty`可以看作是一份"完全拷贝".(本人没有java和netty的经验，只是在查阅资料时发现netty和java-nio的概念完全可以映射到SwiftNIO中，故做此推断)

所以对SwiftNIO中核心概念的介绍会类比于`java`中的`netty`,分为`NIO`和`Swift-Netty`(我自己起的)两部分。

---
### NIO

### SwiftNIO Netty

[1]: https://github.com/apple/swift-nio  "swift-nio"
[2]: https://raw.githubusercontent.com/lbhbrave/blogImageHosting/master/img/libuv.png "libuv.jpg"