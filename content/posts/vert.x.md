---
title: vert.x 实现点对点的通信
date: 2020-10-29 14:24:16
tags: 
- vertx
categories:
- vertx
---

# vert.x

vert.x 已经提供了很便利的 NetClinet，NetServer，这个EventBus Bridge提供的核心能力到底是什么呢？

Server和Client之间采用这种方式进行通信可以让Server端的代码更加清晰明了
Client和Server之间的交互模式也更简单一些：

- 对于request-response，server简单地message.reply即可
- 对于某些广播消息，client向多个server实例广播，或者是server向多个client广播，又或某些client也想接收另外的client向server的广播都非常简单和直观。
- 通过注册地址，client也可以收到感兴趣的消息，不论是server发的，还是client发的

这里有个例子
https://github.com/foxgem/how-to/tree/master/vertx-tcp-eventbridge/src/main/groovy/foxgem

- client向多个bridge send，则只有一个bridge可以收到
- client向多个bridge publish，则都能收到
- server和client之间的request-response
- client注册要接受某地址的消息

vert.x 的EventBus有单机和cluster，bridge和cluster之间的区别又是什么？

EventBus 主要面向的是 verticles（类似actor）间的消息通信，cluster是为了将 EventBus 抽象为分布式的，同时提供了分布式网络下共识的能力

bridge 我理解则是通过跨进程的交互方式，让eventbus成为跨语言通信的 polyglot

EventBus虽然有tcp bridge，但是本意是用做verticles之间的通信用的

并不是用作客户端和服务器端之间的通信用的，正确的通信姿势应该是用vert.x的各种clients & servers

详细使用代码参见测试用例：
https://github.com/vert-x3/vertx-tcp-eventbus-bridge/blob/master/src/test/java/io/vertx/ext/eventbus/bridge/tcp/TcpEventBusBridgeTest.java


# 参考

- https://segmentfault.com/a/1190000008294524 Vert.x TCP EventBus Bridge补遗