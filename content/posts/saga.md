---
title: Saga的意义
date: 2019-04-03 15:06:01
tags: 
- Saga
categories: 
- DDD
---
## 什么是Saga？

Saga 这个名词最早是由Hector Garcia-Molina和Kenneth Salem写的Sagas这篇论文里提出来的，但其实Saga并不是什么新事物，在我们传统的系统设计中，它有个更熟悉的名字——“ProcessManager”，只是换了个马甲，还是干同样的事——组合一组逻辑处理复杂流程。
但它与我们平常理解的“ProgressManager”又有不同，它的提出，最早是是为了解决分布式系统中长时间运行事务(long-running business process)的问题，把单一的transaction按照步骤分成一组若干子transaction，通过补偿机制实现最终一致性。

Saga的概念使得强一致性的分布式事务不再是唯一的解决方案，通过保证事务中每一步都可以一个补偿机制，在发生错误后执行补偿事务来保证系统的可用性和最终一致性。

## Saga的意义，优缺点

Saga通过业务补偿的方式，达到最终一致性

## Saga和2PC、TCC的区别

Saga是柔性事务的实现，最终一致性
2PC是基于XA刚性事务，强一致性
TCC属于业务上的分段提交，Try，confirm，cancel都是对应的一段业务逻辑的操作，先预留资源，预留成功后进行确认，不成功就取消，例如转账先冻结资金，进行一系列的余额各方面的检查，发现符合条件就对账户对应的资金状态改为冻结，确认阶段修改状态为扣除，取消的话就把冻结的资金加回原账户，其对应的数据库的操作每段都是一个完整的事物；2PC是属于数据库层面的，先进行prepare，然后逐个进行commit或者rollback，不和具体业务逻辑挂钩，TCC的应用范围更广，不一定是事物关系数据库，也可能操作的KV数据库，文档数据库，粒度也可以随着具体业务灵活调整，性能更好。

## 参考

- <https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj591569(v%3dpandp.10)>
- <http://edisonxu.com/2017/03/31/axon-saga.html>
