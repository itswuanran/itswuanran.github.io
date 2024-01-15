---
title: 插入死锁的问题
date: 2019-03-17 19:32:06
tags:
- deadlock
categories:
- MySQL
---

一个事务内大量的insert语句导致事务出现死锁的问题，为什么单纯的insert语句会导致死锁呢？
首先，我们数据库有两种锁，一种叫S锁，也就是sharable lock，共享锁，可以理解为读锁就是共享锁；
一种叫X锁，也就是exclusive lock，排它锁。故名思议，排它锁与其他任何锁都不兼容，而共享锁之间是兼容的，可以理解为写锁就是排它锁。
第二个，还有一种锁，叫意向锁，intend lock，意向锁也分为共享锁和排它锁。用于加在所要加S锁或X锁对象的所有父节点上，也就是说你要获取Y（Y代表S或者X）锁，你先要（在父节点）成功获取IY锁。加锁的顺序是从上往下的，任何一个节点加锁失败，事务都处于等待状态。所谓父子节点可以参考以下图：
你可以理解为意向锁总是要加的，但是除非你被加上了S锁或者X锁，意向锁总是能成功的。

另外第三个概念：隐式锁和显式锁，又是令人蛋碎的概念，隐式锁你可以理解为乐观锁，也就是正常来说不加锁或共享锁，但是遇到冲突则加锁或升级为排它锁。显式锁，那就是真的锁上了。不明白为什么总是要用这么晦涩的术语来描述。

OK，言归正传，我们基于上面这些介绍开始分析为什么会insert出现死锁。

先看看这个表，注意token字段是一个唯一索引

```sql
mysql> CREATE TABLE `deadlocktest` (
    ->   `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
    ->   `token` varchar(255) NOT NULL,
    ->   PRIMARY KEY (`id`),
    ->   UNIQUE KEY `ux_token` (`token`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
Query OK, 0 rows affected (0.10 sec)
```
然后，开启三个TRX执行简单的插入操作：

step1：s1-s3 开始，查看锁等待和上锁状态

step2:  s1插入一条记录

```sql
mysql> insert into deadlocktest (token) values ('token1');
Query OK, 1 row affected (0.03 sec)
```
这时的锁、锁等待和事务状态如下：
```sql
mysql> select * from performance_schema.data_locks\G
Empty set (0.00 sec)

mysql> select * from performance_schema.data_lock_waits\G
Empty set (0.00 sec)
```
SHOW ENGINE INNODB STATUS

注意看，表里面并没有显示加锁（这就是传说的隐式锁），但是实际上，s1加了两个锁：一个是表级别的IX锁，一个是行级别的S锁
```sql
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 15922:1079
ENGINE_TRANSACTION_ID: 15922
            THREAD_ID: 51
             EVENT_ID: 15
        OBJECT_SCHEMA: tech
          OBJECT_NAME: deadlocktest
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140497454685400
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
1 row in set (0.00 sec)
```


```sql
mysql> select * from performance_schema.data_locks;
+--------+----------------+-----------------------+-----------+----------+---------------+--------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
| ENGINE | ENGINE_LOCK_ID | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME  | PARTITION_NAME | SUBPARTITION_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA |
+--------+----------------+-----------------------+-----------+----------+---------------+--------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
| INNODB | 15924:1079     |                 15924 |        53 |       13 | tech          | deadlocktest | NULL           | NULL              | NULL       |       140497454689368 | TABLE     | IX        | GRANTED     | NULL      |
| INNODB | 15924:22:5:2   |                 15924 |        53 |       13 | tech          | deadlocktest | NULL           | NULL              | ux_token   |       140497507137048 | RECORD    | S         | WAITING     | 'token1'  |
| INNODB | 15923:1079     |                 15923 |        52 |       13 | tech          | deadlocktest | NULL           | NULL              | NULL       |       140497454687368 | TABLE     | IX        | GRANTED     | NULL      |
| INNODB | 15923:22:5:2   |                 15923 |        52 |       13 | tech          | deadlocktest | NULL           | NULL              | ux_token   |       140497507132440 | RECORD    | S         | WAITING     | 'token1'  |
| INNODB | 15922:1079     |                 15922 |        51 |       15 | tech          | deadlocktest | NULL           | NULL              | NULL       |       140497454685400 | TABLE     | IX        | GRANTED     | NULL      |
| INNODB | 15922:22:5:2   |                 15922 |        52 |       13 | tech          | deadlocktest | NULL           | NULL              | ux_token   |       140497507127832 | RECORD    | X         | GRANTED     | 'token1'  |
+--------+----------------+-----------------------+-----------+----------+---------------+--------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
6 rows in set (0.00 sec)

mysql> select * from performance_schema.data_lock_waits;
Empty set (0.00 sec)

mysql>
```
step3：s2/s3插入相同的token。

这时的锁、锁等待和事务状态如下：

这里第一个关键：事务1，也就是S1的S锁，原来是看不到的锁，变成X锁，这就是所谓的隐式锁转换为显式锁。
第二个：S2和S3，在相同的记录上等待获取S锁。这就是死锁的关键，如果没有这一步就不会死锁。至于为什么，这里先按下不表，我们先解释下为毛一个插入操作会加入一个S锁：因为在事务被唤醒后，需要检测冲突，没错，因为被挂起的事务知道这一行数据被X锁锁住了，一旦事务被唤醒，那么被锁住的数据就可能被更改，所以需要检测冲突。而冲突检测是通过读取是否存在类似的记录实现的，所以这货加了个S锁请求。从上文的兼容性中我们可以知道，S锁之间是兼容的，所以，一旦S2-S3被唤醒，那么他们都可以得到这个锁。

进入关键的步骤第四步：

s1 rollback;

这时，s2(或者s3)能成功，而剩下的一个因为死锁检测被重置。当然，前者能成功就是因为后者被重置了。

s2
```sql
mysql> insert into deadlocktest (token) values ('token1');
Query OK, 1 row affected (14.86 sec)
mysql>
```
s3
```sql
mysql> insert into deadlocktest (token) values ('token1');
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
mysql>
```

顺理成章地，数据库爆出了死锁和成功者的加锁信息：

OK，答案显而易见了。

首先，由于冲突检测，后面的家伙都申请了兼容的S锁，导致他们被唤醒后都获取到了这个S锁，经过冲突检测，他们惊喜的发现可以继续了，然后尝试加上不兼容的锁(IX锁)，于是杯具发生了，要成功获取(IX)锁，都要等待对方先释放不兼容的S锁，于是死锁发生了。

commit就不会失败，因为commit后数据落地，两边拿到S锁发现冲突，自然就插入失败了，没有后续加IX锁的行为，也就没有死锁了



## 另一个死锁案例
```
CREATE TABLE `tt` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL,
  `tid` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_tid` (`tid`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8
```
```
+----+------+-----+
| id | name | tid |
+----+------+-----+
|  1 | a    |   1 |
|  2 | b    |   2 |
|  4 | t    |   4 |
|  5 | c    |   5 |
|  6 | c    |  16 |
|  7 | c    |  17 |
+----+------+-----+
```

gap锁：
简单的理解就是执行 delete from tt where tid = 7，tid存在一个索引，mysql根据索引进行搜索，7不在这些key里面而是位于(5,16)间隙中，对(5,16)这个间隙加的锁就叫做Gap锁。
Insert Intention锁：
插入意向锁，insert语句会对插入的行加一个X锁，但是在插入这个行的过程之前，会设置一个Insert intention锁。

存在Insert Intention 锁时，申请Gap锁是允许的；但是存在Gap锁时，申请Insert Intention锁时是被阻止的。

T1:delete from tt where tid = 7;
T2:delete from tt where tid = 8;
T1:insert into tt values(null,'a',8);
T2:insert into tt values(null,'b',7);

T1 持有了Gap(5,16)的X锁；
T2 申请Gap(5,16)的X锁，该申请被授权，所以T2 持有了Gap(5,16)的X锁。
T1 申请Insert Intention(5,16)的X锁，根据之前讲的互斥关系，由于T2持有Gap(5,16)的X锁，该申请被block。
T2 申请Insert Intention(5,16)的X锁，根据之前讲的互斥关系，由于T1持有Gap(5,16)的X锁，该申请被block。

死锁很明显的出现了，T1与T2都持有一个锁，同时都在等对方释放一个锁。

## 相关链接
- https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html
- http://blog.csdn.net/and1kaney/article/details/51214001
- http://mysql.taobao.org/monthly/2016/01/01/
- https://juejin.im/post/5c774114f265da2d993d9908 记一次神奇的Mysql死锁排查