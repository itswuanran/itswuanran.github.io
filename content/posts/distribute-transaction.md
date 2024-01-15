---
title: 关于分布式事务的文章
date: 2020-10-22 20:26:18
tags:
- CAP
- ACID
- 分布式事务
categories:
- 分布式事务
---

## 如何理解事务中的一致性

单机事务一致性的定义

- https://en.wikipedia.org/wiki/Consistency_(database_systems)#mw-head

可以理解单机一致性是：应用系统从一个正确的状态到另一个正确的状态，而ACID就是说事务能够通过AID来保证这个C的过程，C是目的，AID都是方法，ACID中的一致性是数据库保证所有的约束没有被打破

## 关于分布式事务的文章

- ServiceComb中的数据最终一致性方案
https://github.com/apache/servicecomb-pack

- 基于Sagas思路实现：Sagas相关论文
https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf

- （微服务）分布式事务-最大努力交付 && 消息最终一致性方案
https://segmentfault.com/a/1190000011479826


## 关于分布式事务开源框架
https://github.com/seata/seata

https://github.com/dromara/hmily

https://github.com/dromara/myth

https://github.com/dtm-labs/dtm

## 补充资料

- [事件概念正在重塑分布式系统的未来](http://www.jdon.com/49368) 

- [什么是事件溯源Event Sourcing？](http://www.jdon.com/48501)

- [使用事件和消息队列实现分布式事务（转+补充）](https://www.cnblogs.com/lightdb/p/6618874.html)