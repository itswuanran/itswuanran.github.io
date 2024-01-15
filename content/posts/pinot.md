---
title: Pinot
date: 2023-06-09 12:26:19
tags: 
- "OLAP"
---

- https://github.com/apache/pinot

Pinot is designed to execute real-time OLAP queries with low latency on massive amounts of data and events. In addition to real-time stream ingestion, Pinot also supports batch use cases with the same low latency guarantees. It is suited in contexts where fast analytics, such as aggregations, are needed on immutable data, possibly, with real-time data ingestion. Pinot works very well for querying time series data with lots of dimensions and metrics.

Point被设计为在大量的数据和事件中，执行实时低延迟的OLAP查询。除了实时流的集成外，还提供低延迟的批处理保证，擅长处理。在快速分析的场景中很适合，例如 在不可变的数据以及实时数据做数据聚合，pinot非常擅长处理 有大量维度和指标度量的时序数据查询。

官方有很完整的方案和文档

- [Pinot设计方案](https://docs.pinot.apache.org/developers/design-documents)

Pinot使用了一种新型的索引技术[star-tree-index](https://docs.pinot.apache.org/basics/indexing/star-tree-index) 来解决查询效率的问题，在时间和空间效率上做了很好的平衡，近期遇到的为数不多让人兴奋的项目。但这个项目还是引入了Table等一系列概念，使用SQL来做分析，但不支持join等操作，在运维和可靠性方面上手成本还是会比较高。

同时Pinot在架构上也没有采用 存储和计算分层的思路，针对节点的横向扩展可能会存在成本的问题


## See Also
实时分析的收费产品：https://rockset.com/real-time-analytics-comparison/