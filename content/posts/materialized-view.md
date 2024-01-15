---
title: 物化视图设计模式
date: 2023-06-06 18:33:37
tags:
- "materialized-view"
---

## 背景

最近发现一个小册子，囊括了云时代下软件设计的各种方式方法
- [Cloud Design Patterns: Prescriptive Architecture Guidance for Cloud Applications](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/dn568099(v=pandp.10))

## 物化视图设计模式

- [Materialized View Pattern](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/dn589782(v=pandp.10))


物化视图不仅仅是一个技术名词，更是一种软件设计思想

通过构建物化视图，我们几乎可以满足任何定制化的查询诉求，因为视图就是专门为了查询构建的，由此带来的成本就是我们需要牺牲一定的数据失效性，以及数据同步的复杂性

## 实现视图的考量
- 数据如何同步，跨多个数据源如何支持，同步频次怎么确定
- 历史数据是否需要快照，需不需要加上分区的概念
- 要索引的字段有哪些，以及数据存储计算成本的考量

## 数据仓库
视图和数仓的区别和联系，可以任务我们构建的数仓是一个服务于在线近现实时的查询系统，针对数仓提供的OLAP能力，我们更偏向于业务应用系统，而不是BI系统。