---
title: 对enode设计上的思考
date: 2019-05-23 14:58:08
tags: 
- enode
- DDD
categories:
- DDD
---

## 简介

enode是真正意义上的，聚合根完全独立不相互依赖，聚合之间完全无感知，聚合之间通信完全通过消息，消息流程控制器完全无状态，消息幂等处理自动支持，支持分布式，支持actor，in-memory模式的一个框架，聚合的关系上做到了完全的解耦。

<https://github.com/anruence/enode>

## 核心点

- 事件驱动架构

- 全流程异步

- 聚合根in-memory

- 聚合根并发修改问题规避，保证对同一个聚合的操作路由到同一个分区

- 实现了Saga (Process Manager)

### CQRS如何实现避免资源竞争

对于C端，我们的目标是尽可能的在1s内处理更多的Command，也就是数据写请求。在经典DDD的四层架构中，我们会有一个模式叫工作单元模式，即Unit of Work（简称UoW）模式。通过该模式，我们能在应用层，一次性以事务的方式将当前请求所涉及的多个对象的修改提交到DB。微软的EF实体框架的DbContext就是一个UoW模式的实现。这种做法的好处是，一个请求对多个聚合根的修改，能做到强一致性，因为是事务的。但是这种做法，实际上，没有很好的遵守避开资源竞争的原则。

试想，事务A要修改a1,a2,a3三个聚合根；事务B要修改a2,a3,a4；事务C要修改a3,a4,a5三个聚合根。那这样，我们很容易理解，这三个事务只能串行执行，因为它们要修改相同的资源。比如事务A和事务B都要修改a2,a3这两个聚合根，那同一时刻，只能由一个事务能被执行。同理，事务B和事务C也是一样。如果A,B,C这种事务执行的并发很高，那数据库就会出现严重的并发冲突，甚至死锁。那要如何避免这种资源竞争呢？

### 让一个Command总是只修改一个聚合根

这个做法其实就是缩小事务的范围，确保一个事务一次只涉及一条记录的修改。也就是做到，只有单个聚合根的修改才是事务的，让聚合根成为数据强一致性的最小单位。这样我们就能最大化的实现并行修改。但是你会问，但是我一个请求就是会涉及多个聚合根的修改的，这种情况怎么办呢？在CQRS架构中，有一个东西叫Saga。Saga是一种基于事件驱动的思想来实现业务流程的技术，通过Saga，我们可以用最终一致性的方式最终实现对多个聚合根的修改。对于一次涉及多个聚合根修改的业务场景，一般总是可以设计为一个业务流程，也就是可以定义出要先做什么后做什么。比如以银行转账的场景为例子，如果是按照传统事务的做法，那可能是先开启一个事务，然后让A账号扣减余额，再让B账号加上余额，最后提交事务；如果A账号余额不足，则直接抛出异常，同理B账号如果加上余额也遇到异常，那也抛出异常即可，事务会保证原子性以及自动回滚。也就是说，数据一致性已经由DB帮我们做掉了。

但是，如果是Saga的设计，那就不是这样了。我们会把整个转账过程定义为一个业务流程。然后，流程中会包括多个参与该流程的聚合根以及一个用于协调聚合根交互的流程管理器（ProcessManager，无状态），流程管理器负责响应流程中的每个聚合根产生的领域事件，然后根据事件发送相应的Command，从而继续驱动其他的聚合根进行操作。

转账的例子，涉及到的聚合根有：两个银行账号聚合根，一个交易（Transaction）聚合根，它用于负责存储流程的当前状态，它还会维护流程状态变更时的规则约束；然后当然还有一个流程管理器。转账开始时，我们会先创建一个Transaction聚合根，然后它产生一个TransactionStarted的事件，然后流程管理器响应事件，然后发送一个Command让A账号聚合根做减余额的操作；A账号操作完成后，产生领域事件；然后流程管理器响应事件，然后发送一个Command通知Transaction聚合根确认A账号的操作；确认完成后也会产生事件，然后流程管理器再响应，然后发送一个Command通知B账号做加上余额的操作；后续的步骤就不详细讲了。大概意思我想已经表达了。总之，通过这样的设计，我们可以通过事件驱动的方式，来完成整个业务流程。如果流程中的任何一步出现了异常，那我们可以在流程中定义补偿机制实现回退操作。或者不回退也没关系，因为Transaction聚合根记录了流程的当前状态，这样我们可以很方便的后续排查有状态没有正常结束的转账交易。

## 原则

领域事件 => 事件是聚合根产生的，原则是一个命令只能修改一个聚合根，如果遇到一个请求需要操作多个聚合根时，将请求拆分成命令一个个处理（通过Saga机制）
域 => 问题域，系统要解决什么问题
上下文 => 解决方案，解决域问题的系统

- 如何设计聚合

1. 找出你关心的所有领域概念

2. 先假设所有概念都是实体，且都是聚合根

3. 利用排除法，分析哪些概念没有独立的生命周期，则排除。怎么分析生命周期呢，这样思考，是否有场景会直接创建或者修改该概念，而不是先导航到其他概念再修改这个概念

4. 把排查的概念放在其他的聚合根下形成聚合

- 如何理解界限上下文，界限上下文跟子域有什么关系？
领域过大时候需要分拆子域，子域有分核心子域和支撑子域，通用子域等，按业务划分。
之所以要写个程序，这个程序的最核心的作用(核心域)，有行业或者业务共识的模型(通用子域)，为了让核心域能发挥作用补充相关功能串联整个系统的(支撑子域)

- 是否可以一次事务操作多个聚合根？

这种情况应该避免，如果有人这样做了，需要首先分析这两个聚合根的操作是否一定要一起修改？通常应该允许不一起修改的。
就像下订单，处理订单时，创建订单和减库存可以不在一个事务里一样
虽然原则上是一个事务操作一个聚合根，但在单个上下文内使用领域服务操作两个紧密关联的聚合根也可以。只是取舍问题
只是他的这个上下文架构选型可能用了强一致性事务的基础设施支持， 在一个上下文中使用强一致性，一次事务操作多个聚合也是很常见的

- 一个聚合根可以出现在多个领域服务里不？
聚合根不是放在领域服务里的。
领域服务的作用是协调聚合根，通常不需要领域服务。你问出这个问题问感觉你犯了一个错误，就是你认为聚合根的操作一定要通过领域服务暴露出来。
需求是聚合根，服务是服务，所以，你说把需求做成服务A，让人无法理解，太混乱
领域服务只不过是协调多个聚合的操作而已，如果你这个业务操作只牵涉到一个聚合，那么在应用服务层就行。

- 为什么注册时要加锁？
DDD 业务规则要在领域中体现出来，限制条件要显式的表达出来

Q端的任何更新都应该把聚合根ID和事件版本号作为条件

## 参考资料

- https://github.com/tangxuehua/enode
- http://www.cnblogs.com/netfocus/category/496012.html
- https://www.cnblogs.com/netfocus/p/5180378.html
- https://www.cnblogs.com/netfocus/p/4249237.html