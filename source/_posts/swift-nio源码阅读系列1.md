---
title: swift-nio源码阅读系列1
tags: [swift,backend]
date: 2018-05-11 22:33:34
desc:
---

<!--more-->
# 概要
## swift-nio
*Apple* 今年三月份的时候开源了[swift-nio][1]这个库:

> SwiftNIO is a cross-platform asynchronous event-driven network application framework for rapid development of maintainable high performance protocol servers & clients.   
> It's like Netty, but written for Swift.

所以`swift-nio`有如下几个特点:

#### *Network Application Framework*
`SwiftNIO`服务于高性能的**服务器或客户端协议**,也就是说，`SwiftNIO`是一个通用的网络层*应用*，它使用`异步事件驱动`提供高性能网络处理，服务于上层协议,它不是`node`中的`Express`,`socket.io`这种具体的`http`,`websocket`等协议框架,---,基于`swift-nio`我们可以实现`swift`版的各种应用层框架

#### *Cross-platform*  
`SwiftNio`目标旨在支持所有支持`Swift`的平台，目前支持`linux`和`macOS`

#### *Asynchronous event-driven*

`SwiftNIO`为所有的IO调用实现了异步的形式,针对socket网络IO和一般文件IO采用不同策略实现,这里先引用一下几种常见的I/O模型:
> 同步阻塞（Blocking IO）：最简单的一种IO模型，用户线程在进行IO操作的时候通常是个系统调用，用户线程会由用户空间进入内核空间，内核空间数据包准备好后会将数据拷贝到用户空间，这个时候线程在用户态继续执行。

> 同步非阻塞（Non-blocking IO）：同步非阻塞IO即在同步阻塞的基础之上将socket设置为NONBLOCK。这样用户线程在发起IO操作之后可以立即返回，但是用户线程需要不断轮询来请求数据。

> IO多路复用（IO Multiplexing）：即Reactor设计模式，多路复用模型从流程上和同步阻塞的区别不大，主要区别在于操作系统为用户提供了同时轮询多个IO句柄来查看是否有IO事件的接口（如select），这从根本上允许用户可以使用单个线程来管理多个IO句柄的问题,相较于异步IO,epoll只是通知用户sokcet可以用了,用户需要自己把数据从系统拷贝出来(通过系统调用)

> 异步IO（Asynchronous IO）：即Proactor设计模式。在异步IO模型中，用户不需要去轮询IO事件，然后才进行数据的读取，处理；在异步IO模型中，IO事件就绪的时候，内核会开启一个独立的内核线程去执行执行IO操作，实现真正的异步IO。这个时候用户线程可以直接读取内核线程准备好的数据。  

NIO的结合了上述的同步非阻塞和IO多路复用实现，这样实现了单线程处理大量并发并可以异步的处理结果。其实上面说的异步IO是最好的解决方法，但是目前Linux下的异步IO框架`AIO`并不是很成熟，所以这里依旧使用了传统的`Reactor`模式:
1. 通过将socket设置为`NONBLOCK`,注册到`epoll`中，并设置回调函数
2. 通过`epoll`帮我们把可用的socket取回
3. 执行上述的回调，操作准备就绪的socket，用户开始执行IO操作

`epoll`这些I/O模型只是针对于网络I/O(`socket`,`pipe`),并不能用于磁盘上的文件I/O,这两者之间有一些区别，socket这些IO之所以会发生阻塞是因为它在等socket就绪，也即等待另一端的数据到来,这段过程中,线程是不占用cpu时间片的

但是文件IO并没有就绪这个概念, 它就是上述的同步阻塞模型,当你要从文件读数据的时候,cpu将会一直忙碌的帮你把数据从磁盘拷贝到操作系统，再拷贝到用户空间，只要有cpu资源它就可以立马工作(有可用的线程),因为这个过程是同步的，会阻塞线程中的其他任务,所以为了不影响`epoll`线程分发`socket`事件，使用一个线程池专门处理这些文件IO,其他一些cpu耗时的任务也可以扔到线程池中。    

`SwiftNIO`中的`NIO`指代的就是上述的IO模型,提供了高性能的Socket处理,基于这个`NIO`,`SwiftNIO`又做了一些抽象和封装,从而实现了一个通用的网络层应用
之前有了解Node.js的底层实现,`node`底层依托于`libuv`这个C++库,而`libuv`为node提供了`No blocking IO`的能力,下面是`libuv`的架构图:
<div align=center>
<img src="https://raw.githubusercontent.com/lbhbrave/blogImageHosting/master/img/libuv.png" width="70%" height="70%">
</div>


将图中的几个关键名词换成`SwiftNIO`中的名字,它就是`SwiftNIO`中的`NIO`部分的架构了:
- `uv_io_t`对应`Selector`,封装了不同系统下的IO多路复用模块(不包括图中的`event ports`)，提供统一接口
- `Thread Pool`对应`BlockingIOThreadPool`,线程池的实现
- `File I/O`对应`NonBlockingFileIO`，使用线程池实现的异步文件I/O
- IOCP是Windows下提供的异步I/O接口，和`epoll`这种有本质上的区别，上面已经提到过，目前SwiftNIO还不支持Windows平台

`SwiftNIO`本身可以看作是两部分,一部分是`NIO`,另一部分是网络框架,引用官方的介绍:
> SwiftNIO does not aim to provide high-level solutions like, for example, web frameworks do. Instead, SwiftNIO is focused on providing the low-level building blocks for these higher-level applications. When it comes to building a web application, most users will not want to use SwiftNIO directly: instead, they'll want to use one of the many great web frameworks available in the Swift ecosystem. Those web frameworks, however, may choose to use SwiftNIO under the covers to provide their networking support

接下来会对这两部分的一些核心概念进行介绍

---
### NIO

### SwiftNIO Netty

[1]: https://github.com/apple/swift-nio  "swift-nio"
[2]: https://raw.githubusercontent.com/lbhbrave/blogImageHosting/master/img/libuv.png "libuv.jpg"