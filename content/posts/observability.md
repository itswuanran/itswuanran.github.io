---
title: 可观测性
date: 2020-12-25 14:13:18
categories:
- observability
tags:
- observability
---

## 可观测性

日志（Logging）：日志的职责是记录离散事件，通过这些记录事后分析出程序的行为，譬如曾经调用过什么方法，曾经操作过哪些数据，等等。打印日志被认为是程序中最简单的工作之一，调试问题时常有人会说“当初这里记得打点日志就好了”，可见这就是一项举手之劳的任务。输出日志的确很容易，但收集和分析日志却可能会很复杂，面对成千上万的集群节点，面对迅速滚动的事件信息，面对数以 TB 计算的文本，传输与归集都并不简单。对大多数程序员来说，分析日志也许就是最常遇见也最有实践可行性的“大数据系统”了。

追踪（Tracing）：单体系统时代追踪的范畴基本只局限于栈追踪（Stack Tracing），调试程序时，在 IDE 打个断点，看到的 Call Stack 视图上的内容便是追踪；编写代码时，处理异常调用了 Exception::printStackTrace()方法，它输出的堆栈信息也是追踪。微服务时代，追踪就不只局限于调用栈了，一个外部请求需要内部若干服务的联动响应，这时候完整的调用轨迹将跨越多个服务，同时包括服务间的网络传输信息与各个服务内部的调用堆栈信息，因此，分布式系统中的追踪在国内常被称为“全链路追踪”（后文就直接称“链路追踪”了），许多资料中也称它为“分布式追踪”（Distributed Tracing）。追踪的主要目的是排查故障，如分析调用链的哪一部分、哪个方法出现错误或阻塞，输入输出是否符合预期，等等。

度量（Metrics）：度量是指对系统中某一类信息的统计聚合。譬如，证券市场的每一只股票都会定期公布财务报表，通过财报上的营收、净利、毛利、资产、负债等等一系列数据来体现过去一个财务周期中公司的经营状况，这便是一种信息聚合。Java 天生自带有一种基本的度量，就是由虚拟机直接提供的 JMX（Java Management eXtensions）度量，诸如内存大小、各分代的用量、峰值的线程数、垃圾收集的吞吐量、频率，等等都可以从 JMX 中获得。度量的主要目的是监控（Monitoring）和预警（Alert），如某些度量指标达到风险阈值时触发事件，以便自动处理或者提醒管理员介入。


## OpenTelemetry

- https://opentelemetry.io/

在可观测性领域已经形成了一套规范，在过去，一个物理机器的状态确实可以通过几个监控指标描述，但是随着我们的系统越来越复杂，我们的观测对象正渐渐的从「Infrastructure」转到「应用」，观察行为本身从「Monitoring（监控）」到「Observability（观测）」。虽然看上去这两者只是文字上的差别，但是请仔细思考背后的含义。

## 链接
- https://zhuanlan.zhihu.com/p/83654617
- https://zhuanlan.zhihu.com/p/108485161