---
title: 什么是CQRS
date: 2018-05-08 16:54:38
tags: 
- cqrs
categories:
- DDD
---
## CQRS简介
CQRS全称为Command Query Responsibility Segregation，顾名思义“命令与查询职责分离”。“职责分离”我们理解，但怎么区分“命令”与“查询”，他们的职责又分别是什么？
命令与查询的根本区别在于，是否改变数据的状态。例如增、删、改操作即归属于“命令”，因为这些操作会导致数据被修改；而查询操作只求返回结果，并不修改数据，所以归属于“查询”（查询归属于“查询”，好吧，听上去像废话）。另一个区别在于，“命令”操作不需要返回值（当然我们在编码时需要有返回来告诉我们修改是否成功），而“查询”需要。

简单来说，CQRS作用在于把数据的读和写分离。读写操当然是隔离的，这里的分离相对的是传统编程中一视同仁的编写CRUD(Create, Read, Update, Delete，增删改查)接口代码。CQRS主张对“读”和“写”的接口做不同的设计，编码和优化。而之所以要做这样的分离，原因有以下两点：

- 在许多业务场景中，数据的读和写的次数是不平衡，可能上千次的读操作才对应一次写操作，比如机票余票信息的查询和更新。所以把读和写操作分开能够有针对性的分别优化它们。例如提高程序的scalability，scalability意味着我们能够在部署程序时，给读操作和写操作部署不同数量的线上实例来满足实际的需求。
- 通常来说，“写”操作比“读”操作更加复杂，无论是业务逻辑方面还是技术方面。例如写操作首先要验证数据的合法性和完整性，存储中会涉及事务，存储过程，同时还要考虑数据冗余，数据同步；而读操作则完全没有类似的要求。把读和写分离相当于隔离了复杂操作，分离便于我们更好的独立维护它们，

## 参考
- https://zhuanlan.zhihu.com/p/25383827
- https://msdn.microsoft.com/en-us/library/jj554200.aspx
- https://www.codeproject.com/Articles/555855/Introduction-to-CQRS
- http://www.cnblogs.com/netfocus/p/5184182.html
- http://www.techweb.com.cn/network/system/2017-07-07/2553563.shtml
- https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs
- http://www.uml.org.cn/zjjs/201609221.asp
- http://edisonxu.com/2017/03/30/axon-cqrs-example.html

