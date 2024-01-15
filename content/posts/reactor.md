---
title: reactor模式
date: 2018-03-29 18:18:58
categories:
- netty
tags:
- reactor
---
## 什么是reactor设计模式

reactor设计模式，是一种基于事件驱动的设计模式。Reactor框架是ACE各个框架中最基础的一个框架，其他框架都或多或少地用到了Reactor框架。 在事件驱动的应用中，将一个或多个客户的服务请求分离（demultiplex）和调度（dispatch）给应用程序。在事件驱动的应用中，同步地、有序地处理同时接收的多个服务请求。 
reactor模式与外观模式有点像。不过，观察者模式与单个事件源关联，而反应器模式则与多个事件源关联 。当一个主体发生改变时，所有依属体都得到通知。

反应器设计模式(Reactor pattern)是一种为处理并发服务请求，并将请求提交到一个或者多个服务处理程序的事件设计模式。当客户端请求抵达后，服务处理程序使用多路分配策略，由一个非阻塞的线程来接收所有的请求，然后派发这些请求至相关的工作线程进行处理。 Reactor模式主要包含下面几部分内容。

- 初始事件分发器(Initialization Dispatcher)：用于管理Event Handler，定义注册、移除EventHandler等。它还作为Reactor模式的入口调用Synchronous Event Demultiplexer的select方法以阻塞等待事件返回，当阻塞等待返回时，根据事件发生的Handle将其分发给对应的Event Handler处理，即回调EventHandler中的handle_event()方法

- 同步（多路）事件分离器(Synchronous Event Demultiplexer)：无限循环等待新事件的到来，一旦发现有新的事件到来，就会通知初始事件分发器去调取特定的事件处理器。

- 系统处理程序(Handles)：操作系统中的句柄，是对资源在操作系统层面上的一种抽象，它可以是打开的文件、一个连接(Socket)、Timer等。由于Reactor模式一般使用在网络编程中，因而这里一般指Socket Handle，即一个网络连接（Connection，在Java NIO中的Channel）。这个Channel注册到Synchronous Event Demultiplexer中，以监听Handle中发生的事件，对ServerSocketChannnel可以是CONNECT事件，对SocketChannel可以是READ、WRITE、CLOSE事件等。

- 事件处理器(Event Handler)： 定义事件处理方法，以供Initialization Dispatcher回调使用。

对于Reactor模式，可以将其看做由两部分组成，一部分是由Boss组成，另一部分是由worker组成。Boss就像老板一样，主要是拉活儿、谈项目，一旦Boss接到活儿了，就下发给下面的work去处理。也可以看做是项目经理和程序员之间的关系。

## 标准定义两种I/O多路复用模式：
Reactor和Proactor一般地，I/O多路复用机制都依赖于一个事件多路分离器(Event Demultiplexer)。
分离器对象可将来自事件源的I/O事件分离出来，并分发到对应的read/write事件处理器(Event Handler)。
开发人员预先注册需要处理的事件及其事件处理器（或回调函数）；
事件分离器负责将请求事件传递给事件处理器。两个与事件分离器有关的模式是Reactor和Proactor。
Reactor模式采用同步IO，而Proactor采用异步IO。

在Reactor中，事件分离器负责等待文件描述符或socket为读写操作准备就绪，然后将就绪事件传递给对应的处理器，最后由处理器负责完成实际的读写工作。
而在Proactor模式中，处理器--或者兼任处理器的事件分离器，只负责发起异步读写操作。IO操作本身由操作系统来完成。传递给操作系统的参数需要包括用户定义的数据缓冲区地址和数据大小，操作系统才能从中得到写出操作所需数据，或写入从socket读到的数据。事件分离器捕获IO操作完成事件，然后将事件传递给对应处理器。比如，在windows上，处理器发起一个异步IO操作，再由事件分离器等待IOCompletion事件。典型的异步模式实现，都建立在操作系统支持异步API的基础之上，我们将这种实现称为“系统级”异步或“真”异步，因为应用程序完全依赖操作系统执行真正的IO工作。

举个例子，将有助于理解Reactor与Proactor二者的差异，以读操作为例（类操作类似）。
在Reactor中实现读：
- 注册读就绪事件和相应的事件处理器
- 事件分离器等待事件
- 事件到来，激活分离器，分离器调用事件对应的处理器。
- 事件处理器完成实际的读操作，处理读到的数据，注册新的事件，然后返还控制权。

在Proactor中实现读：
- 处理器发起异步读操作（注意：操作系统必须支持异步IO）。在这种情况下，处理器无视IO就绪事件，它关注的是完成事件。
- 事件分离器等待操作完成事件。
- 在分离器等待过程中，操作系统利用并行的内核线程执行实际的读操作，并将结果数据存入用户自定义缓冲区，最后通知事件分离器读操作完成。
- 事件分离器呼唤处理器。
- 事件处理器处理用户自定义缓冲区中的数据，然后启动一个新的异步操作，并将控制权返回事件分离器。
可以看出，两个模式的相同点，都是对某个IO事件的事件通知(即告诉某个模块，这个IO操作可以进行或已经完成)。在结构上，两者也有相同点：demultiplexor负责提交IO操作(异步)、查询设备是否可操作(同步)，然后当条件满足时，就回调handler；不同点在于，异步情况下(Proactor)，当回调handler时，表示IO操作已经完成；同步情况下(Reactor)，回调handler时，表示IO设备可以进行某个操作(can read or can write)。2、通俗理解使用Proactor框架和Reactor框架都可以极大的简化网络应用的开发，但它们的重点却不同。Reactor框架中用户定义的操作是在实际操作之前调用的。比如你定义了操作是要向一个SOCKET写数据，那么当该SOCKET可以接收数据的时候，你的操作就会被调用；而Proactor框架中用户定义的操作是在实际操作之后调用的。比如你定义了一个操作要显示从SOCKET中读入的数据，那么当读操作完成以后，你的操作才会被调用。Proactor和Reactor都是并发编程中的设计模式。在我看来，他们都是用于派发/分离IO操作事件的。这里所谓的IO事件也就是诸如read/write的IO操作。"派发/分离"就是将单独的IO事件通知到上层模块。两个模式不同的地方在于，Proactor用于异步IO，而Reactor用于同步IO。

## 参考
- https://blog.csdn.net/pistolove/article/details/53152708
- https://blog.csdn.net/u010168160/article/details/53019039
- https://www.zhihu.com/question/24322387
- https://www.zhihu.com/question/26943938/answer/68773398
- https://github.com/reactor/reactor