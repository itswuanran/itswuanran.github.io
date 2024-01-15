---
title: RocketMQ中的核心概念
date: 2019-01-30 19:39:35
tags:
- RocketMQ
categories:
- RocketMQ
---

## RocketMQ 是什么？

RocketMQ是一款分布式、队列模型的消息中间件，具有以下特点：

- 能够保证严格的消息顺序
- 提供丰富的消息拉取模式
- 高效的订阅者水平扩展能力
- 实时的消息订阅机制
- 亿级消息堆积能力

## 基本组件

RocketMQ服务端的组件有三个，NameServer，Broker，FilterServer（可选，部署于和Broker同一台机器）

下面分别介绍三个组件：

## Name Server

Name Server是RocketMQ的寻址服务。用于把Broker的路由信息做聚合。用户端依靠Name Server决定去获取对应topic的路由信息，从而决定对哪些Broker做连接。

Name Server是一个几乎无状态的结点，Name Server之间采取share-nothing的设计，互不通信。

对于一个Name Server集群列表，客户端连接Name Server的时候，只会选择随机连接一个结点，以做到负载均衡。

Name Server所有状态都从Broker上报而来，本身不存储任何状态，所有数据均在内存。

如果中途所有Name Server全都挂了，影响到路由信息的更新，不会影响和Broker的通信。

## Broker

Broker是处理消息存储，转发等处理的服务器。

Broker以group分开，每个group只允许一个master，若干个slave。
只有master才能进行写入操作，slave不允许。
slave从master中同步数据。同步策略取决于master的配置，可以采用同步双写，异步复制两种。
客户端消费可以从master和slave消费。在默认情况下，消费者都从master消费，在master挂后，客户端由于从Name Server中感知到Broker挂机，就会从slave消费。
Broker向所有的NameServer结点建立长连接，注册Topic信息。

## Filter Server（可选）

RocketMQ可以允许消费者上传一个Java类给Filter Server进行过滤。

Filter Server只能起在Broker所在的机器
可以有若干个Filter Server进程
拉取消息的时候，消息先经过Filter Server，Filter Server靠上传的Java类过滤消息后才推给Consumer消费。
客户端完全可以消费消息的时候做过滤，不需要Filter Server
FilterServer存在的目的是用Broker的CPU资源换取网卡资源。因为Broker的瓶颈往往在网卡，而且CPU资源很闲。在客户端过滤会导致无需使用的消息在占用网卡资源。
使用 Java class上传作为过滤表达式是一个双刃剑——一方面方便了应用的过滤操作且节省网卡资源，另一方面也带来了服务器端的安全风险。这就需要应用来保证过滤代码安全——例如在过滤程序里尽可能不做申请大内存，创建线程等操作。避免 Broker 服务器资源泄漏。

## 核心概念

## Producer

生产者。发送消息的客户端角色。发送消息的时候需要指定Topic。

## Consumer

消费者。消费消息的客户端角色。通常是后台处理异步消费的系统。 RocketMQ中Consumer有两种实现：PushConsumer和PullConsumer。

## PushConsumer

推送模式（虽然RocketMQ使用的是长轮询）的消费者。消息的能及时被消费。使用非常简单，内部已处理如线程池消费、流控、负载均衡、异常处理等等的各种场景。

## PullConsumer

拉取模式的消费者。应用主动控制拉取的时机，怎么拉取，怎么消费等。主动权更高。但要自己处理各种场景。

## 生产组 Producer Group

标识发送同一类消息的Producer，通常发送逻辑一致。发送普通消息的时候，仅标识使用，并无特别用处。若事务消息，如果某条发送某条消息的producer-A宕机，使得事务消息一直处于PREPARED状态并超时，则broker会回查同一个group的其 他producer，确认这条消息应该commit还是rollback。但开源版本并不支持事务消息。

## 消费组 Consumer Group

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
```

RocketMQ中，这步操作就是初始化一个please_rename_unique_group_name消费组。消费组屏蔽了consumer和broker数量的关系，不需要自定制。
标识一类Consumer的集合名称，这类Consumer通常消费一类消息，且消费逻辑一致。同一个Consumer Group下的各个实例将共同消费topic的消息，起到负载均衡的作用。

消费进度以Consumer Group为粒度管理，不同Consumer Group之间消费进度彼此不受影响，即消息A被Consumer Group1消费过，也会再给Consumer Group2消费。

注： RocketMQ要求同一个Consumer Group的消费者必须要拥有相同的注册信息，即必须要听一样的topic(并且tag也一样)。

## Topic

标识一类消息的逻辑名字，消息的逻辑管理单位。无论消息生产还是消费，都需要指定Topic。

## Topic中Tag的概念

RocketMQ支持给在发送的时候给topic打tag，同一个topic的消息虽然逻辑管理是一样的。但是消费topic1的时候，如果你订阅的时候指定的是tagA，那么tagB的消息将不会投递。tag的目的是作为一类类型字段。对业务而言有更好的扩展性。

## Message Queue

简称Queue或Q。消息物理管理单位。一个Topic将有若干个Q。若Topic同时创建在不通的Broker，则不同的broker上都有若干Q，消息将物理地存储落在不同Broker结点上，具有水平扩展的能力。

无论生产者还是消费者，实际的生产和消费都是针对Q级别。例如Producer发送消息的时候，会预先选择（默认轮询）好该Topic下面的某一条Q地发送；Consumer消费的时候也会负载均衡地分配若干个Q，只拉取对应Q的消息。

每一条message queue均对应一个文件，这个文件存储了实际消息的索引信息。并且即使文件被删除，也能通过实际纯粹的消息文件（commit log）恢复回来。

### Offset

RocketMQ中，有很多offset的概念。但通常我们只关心暴露到客户端的offset。一般我们不特指的话，就是指逻辑Message Queue下面的offset。

注： 逻辑offset的概念在RocketMQ中字面意思实际上和真正的意思有一定差别，这点在设计上显得有点混乱。祥见下面的解释。

可以认为一条逻辑的message queue是无限长的数组。一条消息进来下标就会涨1,而这个数组的下标就是offset。

### max offset

字面上可以理解为这是标识message queue中的max offset表示消息的最大offset。但是从源码上看，这个offset实际上是最新消息的offset+1，即：下一条消息的offset。

### min offset

标识现存在的最小offset。而由于消息存储一段时间后，消费会被物理地从磁盘删除，message queue的min offset也就对应增长。这意味着比min offset要小的那些消息已经不在broker上了，无法被消费。

### consumer offset

字面上，可以理解为标记Consumer Group在一条逻辑Message Queue上，消息消费到哪里即消费进度。但从源码上看，这个数值是消费过的最新消费的消息offset+1，即实际上表示的是下次拉取的offset位置。

消费者拉取消息的时候需要指定offset，broker不主动推送消息， offset的消息返回给客户端。

consumer刚启动的时候会获取持久化的consumer offset，用以决定从哪里开始消费，consumer以此发起第一次请求。

每次消息消费成功后，这个offset在会先更新到内存，而后定时持久化。在集群消费模式下，会同步持久化到broker，而在广播模式下，则会持久化到本地文件。

集群消费
消费者的一种消费模式。一个Consumer Group中的各个Consumer实例分摊去消费消息，即一条消息只会投递到一个Consumer Group下面的一个实例。

实际上，每个Consumer是平均分摊Message Queue的去做拉取消费。例如某个Topic有3条Q，其中一个Consumer Group 有 3 个实例（可能是 3 个进程，或者 3 台机器），那么每个实例只消费其中的1条Q。

而由Producer发送消息的时候是轮询所有的Q,所以消息会平均散落在不同的Q上，可以认为Q上的消息是平均的。那么实例也就平均地消费消息了。

这种模式下，消费进度的存储会持久化到Broker。

## 广播消费

消费者的一种消费模式。消息将对一个Consumer Group下的各个Consumer实例都投递一遍。即即使这些 Consumer 属于同一个Consumer Group，消息也会被Consumer Group 中的每个Consumer都消费一次。

实际上，是一个消费组下的每个消费者实例都获取到了topic下面的每个Message Queue去拉取消费。所以消息会投递到每个消费者实例。

这种模式下，消费进度会存储持久化到实例本地。

## 顺序消息

消费消息的顺序要同发送消息的顺序一致。由于Consumer消费消息的时候是针对Message Queue顺序拉取并开始消费，且一条Message Queue只会给一个消费者（集群模式下），所以能够保证同一个消费者实例对于Q上消息的消费是顺序地开始消费（不一定顺序消费完成，因为消费可能并行）。

在RocketMQ中，顺序消费主要指的是都是Queue级别的局部顺序。这一类消息为满足顺序性，必须Producer单线程顺序发送，且发送到同一个队列，这样Consumer就可以按照Producer发送的顺序去消费消息。

生产者发送的时候可以用MessageQueueSelector为某一批消息（通常是有相同的唯一标示id）选择同一个Queue，则这一批消息的消费将是顺序消息（并由同一个consumer完成消息）。或者Message Queue的数量只有1，但这样消费的实例只能有一个，多出来的实例都会空跑。

## 普通顺序消息

顺序消息的一种，正常情况下可以保证完全的顺序消息，但是一旦发生异常，Broker宕机或重启，由于队列总数发生发化，消费者会触发负载均衡，而默认地负载均衡算法采取哈希取模平均，这样负载均衡分配到定位的队列会发化，使得队列可能分配到别的实例上，则会短暂地出现消息顺序不一致。

如果业务能容忍在集群异常情况（如某个 Broker 宕机或者重启）下，消息短暂的乱序，使用普通顺序方式比较合适。

## 严格顺序消息

顺序消息的一种，无论正常异常情况都能保证顺序，但是牺牲了分布式 Failover 特性，即 Broker集群中只要有一台机器不可用，则整个集群都不可用，服务可用性大大降低。

如果服务器部署为同步双写模式，此缺陷可通过备机自动切换为主避免，不过仍然会存在几分钟的服务不可用。（依赖同步双写，主备自动切换，自动切换功能目前并未实现）

目前已知的应用只有数据库 binlog 同步强依赖严格顺序消息，其他应用绝大部分都可以容忍短暂乱序，推荐使用普通的顺序消息

## 消息队列的推模式和拉模式

### 拉模式

消费者“拉取”模式，一般采用轮询的方式进行消息拉取。如果频繁的进行轮询拉取，一方面会浪费 CPU 的资源，另一方面会耗费大量的网络流量。此外，对于频繁更新的数据还会造成数据无法实时一致性。（采用拉取方式，会增加消息的延迟，即消息到达消费者的时间有点长(adds significant latency per message)）因此，对于实时性不强的场景，可以考虑使用消费者“拉取”模式。

#### 长轮询

说到Long Polling（长轮询），必然少不了提起Polling（轮询），这都是拉模式的两种方式。
Polling是指不管服务端数据有无更新，客户端每隔定长时间请求拉取一次数据，可能有更新数据返回，也可能什么都没有。
Long Polling原理也很简单，相比Polling，客户端发起Long Polling，此时如果服务端没有相关数据，会hold住请求，直到服务端有相关数据，或者等待一定时间超时才会返回。返回后，客户端又会立即再次发起下一次Long Polling。这种方式也是对拉模式的一个优化，解决了拉模式数据通知不及时，以及减少了大量的无效轮询次数。（所谓的hold住请求指的服务端暂时不回复结果，保存相关请求，不关闭请求连接，等相关数据准备好，写会客户端。）

### 推模式

生产者“推送”模式，一般采用“发布-订阅”的方式进行消息推送，“推送”模式的好处在于，实时性较高，消费者可以较快的获取到更新数据，并且可以避免消费者空轮询拉取导致消耗资源。但是，我们需要确保可靠事件投递并且消息队列确保事件传递至少一次，否则会造成消息丢失。因此，实时性很强的场景，可以考虑使用生产者“推送”模式。

## 参考

- [RocketMQ高性能之底层存储设计](https://mp.weixin.qq.com/s/yd1oQefnvrG1LLIoes8QAg)
- [Long Polling长轮询详解](https://www.jianshu.com/p/d3f66b1eb748)
- <https://blog.csdn.net/javahongxi/article/details/72956608>
- <http://jaskey.github.io/blog/2016/12/15/rocketmq-concept/>
- [一文讲透Apache RocketMQ技术精华](https://mp.weixin.qq.com/s/Efw5pXrptWPfCDQI8rc_xg)