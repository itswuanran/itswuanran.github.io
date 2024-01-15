---
title: kafka的关键概念
date: 2019-03-25 18:05:47
tags:
- kafka
categories:
- kafka
---
## kafka如何实现高吞吐？
高吞吐是 Kafka 需要实现的核心目标之一，为此 kafka 做了以下一些设计：
1. 数据磁盘持久化：消息不在内存中 Cache ，直接写入到磁盘，充分利用磁盘的顺序读写性能。
直接使用 Linux 文件系统的 Cache ，来高效缓存数据。
文件追加写
2. zero-copy：减少 IO 操作步骤
采用 Linux Zero-Copy 提高发送性能。
- 传统的数据发送需要发送 4 次上下文切换。
- 采用 sendfile 系统调用之后，数据直接在内核态交换，系统上下文切换减少为2次。根据测试结果，可以提高60%的数据发送性能。
避免二次拷贝以及内核态到用户态的切换，详情见：https://developer.ibm.com/articles/j-zerocopy/

3. 数据批量发送
4. 数据压缩
5. Topic 划分为多个 Partition ，提高并行度。
数据在磁盘上存取代价为 O(1)。
Kafka 以 Topic 来进行消息管理，每个 Topic 包含多个 Partition ，每个 Partition 对应一个逻辑 log ，有多个 segment 文件组成。
每个 segment 中存储多条消息（见下图），消息 id 由其逻辑位置决定，即从消息 id 可直接定位到消息的存储位置，避免 id 到位置的额外映射。
每个 Partition 在内存中对应一个 index ，记录每个 segment 中的第一条消息偏移。
发布者发到某个 Topic 的消息会被均匀的分布到多个 Partition 上（随机或根据用户指定的回调函数进行分布），Broker 收到发布消息往对应 Partition 的最后一个 segment 上添加该消息。
当某个 segment上 的消息条数达到配置值或消息发布时间超过阈值时，segment上 的消息会被 flush 到磁盘，只有 flush 到磁盘上的消息订阅者才能订阅到，segment 达到一定的大小后将不会再往该 segment 写数据，Broker 会创建新的 segment 文件。

6. Broker NIO异步消息处理，实现了IO线程与业务线程分离。

## Kafka 的副本机制是怎么样的？
- https://blog.csdn.net/u013256816/article/details/71091774

## 什么是 Kafka 事务？
- https://blog.csdn.net/ransom0512/article/details/78840042

## Kafka 如何保证消息的顺序性？
Kafka 本身，并不像 RocketMQ 一样，提供顺序性的消息。所以，提供的方案，都是相对有损的。如下：
这里的顺序消息，我们更多指的是，单个 Partition 的消息，被顺序消费。
方式一，Consumer ，对每个 Partition 内部单线程消费，单线程吞吐量太低，一般不会用这个。
方式二，Consumer ，拉取到消息后，写到N个内存 queue，具有相同 key 的数据都到同一个内存 queue 。然后，对于N个线程，每个线程分别消费一个内存 queue 即可，这样就能保证顺序性。
这种方式，相当于对【方式一】的改进，将相同 Partition 的消息进一步拆分，保证相同 key 的数据消费是顺序的。
不过这种方式，消费进度的更新会比较麻烦。
当然，实际情况也不太需要考虑消息的顺序性，基本没有业务需要。

## 参考
- [为什么Kafka这么快？](https://mp.weixin.qq.com/s/pzVS7r3QaQPFwob-fY8b4A)
- [震惊了！原来这才是 Kafka！（多图+深入）](https://mp.weixin.qq.com/s/d9KIz0xvp5I9rqnDAlZvXw)
- [零拷贝](https://github.com/javagrowing/JGrowing/blob/master/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/IO/%E8%B5%B0%E8%BF%9B%E7%A7%91%E5%AD%A6%E4%B9%8B%E6%8F%AD%E5%BC%80%E7%A5%9E%E7%A7%98%E7%9A%84%E9%9B%B6%E6%8B%B7%E8%B4%9D.md)
- [Kafka源码系列之kafka如何实现高性能读写的](https://mp.weixin.qq.com/s/ssJV6wnH7fE2arZ07_Og4w)
- https://blog.csdn.net/csdn_9527666/article/details/82709198
- https://blog.csdn.net/yjh314/article/details/77568580
- http://www.iocoder.cn/Kafka/message-format/
- [你需要知道的kafka](https://juejin.im/post/5b573eafe51d45198f5c80a4)
- [再谈基于 Kafka 和 ZooKeeper 的分布式消息队列原理](https://gitbook.cn/books/5bc446269a9adf54c7ccb8bc/index.html)