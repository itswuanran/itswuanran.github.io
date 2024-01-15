---
title: Redis一致性检测调研
date: 2019-03-11 19:16:43
tags: 
- Redis
categories:
- Redis
---

## Mysql
1、Mysql一致性校验工具(pt-table-checksum)：
https://www.percona.com/doc/percona-toolkit/LATEST/pt-table-checksum.html

binlog replication的校验。对应于redis就是aof。

2、Mysql一致性校验方案(Baron Schwartz)：

https://www.xaprb.com/blog/2007/04/07/how-to-know-if-a-mysql-slave-is-identical-to-its-master/

3、Mysql一致性校验方案工具(mysqlrplsyn）：

https://docs.oracle.com/cd/E17952_01/mysql-utilities-1.6-en/utils-task-rplsync.html

1与3类似，但对redis不适用，因为它们都需要：lock tables，perform then checksum，get information about the master status.

而redis只有一个db，这样做，就不允许期间有写入了。另外，这样的工具，并不能保证结果100%正确，还是有可能会失败的。

## PostgreSQL
1、9.3+版本新增了checksum：

https://wiki.postgresql.org/wiki/What's_new_in_PostgreSQL_9.3#Data_Checksums

缺点：

a. 只能在集群模式下使用。

b. 开启该功能会导致性能损耗。

c. 必须在启动时开启，不支持动态调整。

2、官方回复：

https://www.postgresql-archive.org/replication-consistency-checking-td5852406.html

Stream Replication已经保证了数据的一致性。

反对者：mysql有replication synchronization问题，需要一致性校验工具，但Postgres没有该问题，根本不需要一致性校验工具。

DB2的HADR（similar to PG replication）也是不需要一致性校验工具的。

支持者：Postgres曾经发生过主从不一致的问题：

https://wiki.postgresql.org/wiki/Nov2013ReplicationIssue

https://thebuild.com/presentations/worst-day-fosdem-2014.pdf

解决该问题的方案一般有2种：

a. checksum.（比较重，需要锁住，防止新写入）

b. Keep things monitored. (比较轻)

redis可以在准备写aof那里嵌入一个函数：

a. 针对写入流，做checksum。(db级别）

b. 将写入流，发送给监听的client。（redis只提供数据，由第三方进行具体的一致性校验，以及查找不一致key的工作）

## Oracle
1、Oracle VeriData：

http://www.oracle.com/us/products/middleware/059493.pdf

介绍了Oracle VeriData的功能与特性，没有具体设计。我们的最终实现效果可以参考这个。

## Redislabs
1、https://redislabs.com/blog/fast-and-efficient-parallelized-comparison-of-redis-databases/

主要是用SCAN系列命令，进行比较。缺点：1）对redis性能有影响；2）假如SCAN过程中key更新了，可能结果是不一致的。

用于比较的python脚本：https://gist.github.com/yoav-steinberg/10589500