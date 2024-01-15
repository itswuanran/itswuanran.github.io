---
title: 理解Raft
date: 2018-02-24 10:18:58
tags: 
- raft
categories:
- 分布式
---

## 什么是Raft？

Raft是用于管理日志复制的一致性算法。它的性能和功能跟Paxos一样，但它比Paxos更易于理解并且更易于实现。

Raft将一致性算法分成几个部分：

> 领导选举，即Leader Election。
> 日志复制，即Log Replication。
> 安全性，即Safety。
> 减少状态（state space reduction）

Raft的新特性：

强领导者（strong leader）：Raft 使用一种其他算法更强的领导形式。eg：日志条目只从领导者发送向其他服务器。这样就简化了对日志复制的管理。

领导选取（leader selection）：Raft 使用随机定时器来选取领导者。这种方式仅仅是在所有算法都需要实现的心跳极值上增加了一点变化，在解决冲突时更简单快速

成员变化（Membership change）：Raft 为了调整集群中成员关系使用了新的联合一致性（joint consensus）的方法，这种方法中大多数不同配置的机器在转换关系的时候会交迭（overlap）。这使得在配置改变的时候，集群能够继续操作。
除此之外，Raft还采用一种新的机制-Overlapping Majorities，用于动态改变集群成员。

## Why Raft？

论文中说Paxos算法是所有一致性算法的起点，但是难以理解、实现困难。
Raft的初衷就是设计一个既利于理解又易于实现的一致性算法，Raft目标：

- 必须提供完整、实际的基础，减少开发者工作，方便系统构建；
- 必须在所有情况下都保证安全可用；
- 支持高效的常规操作；
- 最重要的目标：易于理解；
- 它必须能让开发者对算法有一个直观的认识，方便扩展开发(在实际开发中必不可少)；

## 一致性

它是指多个服务器在状态达成一致，但是在一个分布式系统中，因为各种意外可能，有的服务器可能会崩溃或变得不可靠，它就不能和其他服务器达成一致状态。这样就需要一种Consensus协议，一致性协议是为了确保容错性，也就是即使系统中有一两个服务器当机，也不会影响其处理过程。

## 动画演示

- http://thesecretlivesofdata.com/raft/

## 各语言实现

- https://github.com/Waqee/Raft-php
- https://github.com/etcd-io/etcd

## 参考文章

- http://www.infoq.com/cn/articles/raft-paper
- https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md
- https://raft.github.io/
- https://raft.github.io/raft.pdf
- https://www.zhihu.com/question/36648084