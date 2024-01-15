---
title: 偶然看到的面试题收集
date: 2018-03-22 14:45:49
tags:
- interview
categories:
- interview
---
## 面试题

## netty为什么需要IO线程池和业务线程池分开与如何实现
- https://blog.csdn.net/x5fnncxzq4/article/details/82977562
- https://www.cnblogs.com/my_life/articles/5537972.html
- https://www.cnblogs.com/stateis0/p/9062151.html

## 快速替换10亿条标题中的5万个敏感词，有哪些解决思路？
把所有敏感词建立一个 AC 自动机，然后每行标题都在自动机上跑一遍就行了。

- https://www.zhihu.com/question/53009346
- http://blog.csdn.net/creatorx/article/details/71100840
- http://blog.csdn.net/u013371163/article/details/60469534
- https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm
- https://en.wikipedia.org/wiki/Tarjan's_off-line_lowest_common_ancestors_algorithm

## 淘宝面试题：有一个一亿节点的树，现在已知两个点，找这两个点的共同的祖先.有什么好算法呢？
- https://www.zhihu.com/question/19957473

## Java类加载机制
- https://segmentfault.com/a/1190000005608960
- https://www.ibm.com/developerworks/cn/java/j-lo-classloader/

## 两链表数字加和
大数相加的思路
- https://blog.csdn.net/Boring_Wednesday/article/details/80386804

## 为LRU Cache设计一个数据结构
- https://www.cnblogs.com/springfor/p/3869393.html
- https://www.cnblogs.com/dolphin0520/p/3741519.html

## MySQL B+树特性
- https://blog.csdn.net/v_JULY_v/article/details/6530142

## 进程间的通信方式
- https://www.cnblogs.com/wust221/p/5414839.html
- https://blog.csdn.net/wh_sjc/article/details/70283843

## tcp可靠传输的实现
- https://www.cnblogs.com/wxgblogs/p/5612066.html
- https://blog.csdn.net/wyn126/article/details/80472411

## 线程池相关
- https://www.jianshu.com/p/9710b899e749

### 单机上一个线程池正在处理服务，如果忽然断电了怎么办（正在处理和阻塞队列里的请求怎么处理）？
> 我感觉实现思路和MySQL的redo，undo功能很相似，我们可以对正在处理和阻塞队列的任务做事物管理或者对阻塞队列中的任务持久化处理，并且当断电或者系统崩溃，操作无法继续下去的时候，可以通过回溯日志的方式来撤销正在处理的已经执行成功的操作。然后重新执行整个阻塞队列。
即：阻塞队列持久化，正在处理事物控制。断电之后正在处理的回滚，日志恢复该次操作。服务器重启后阻塞队列中的数据再加载

### 为什么要使用线程池？

- 降低资源消耗：通过重用已经创建的线程来降低线程创建和销毁的消耗
- 提高响应速度：任务到达时不需要等待线程创建就可以立即执行
- 提高线程的可管理性：线程池可以统一管理、分配、调优和监控

### 线程池有什么作用？

### 说说几种常见的线程池及使用场景。
- newFixedThreadPool（固定大小的线程池）
- newSingleThreadExecutor（单线程线程池）
- newCachedThreadPool（可缓存线程的线程池）
- newScheduledThreadPool

### 线程池都有哪几种工作队列？
- ArrayBlockingQueue （有界队列）：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
- LinkedBlockingQueue （无界队列）：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
- SynchronousQueue（同步队列）: 一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
- DelayQueue（延迟队列）：一个任务定时周期的延迟执行的队列。根据指定的执行时间从小到大排序，否则根据插入到队列的先后排序。
PriorityBlockingQueue（优先级队列）: 一个具有优先级得无限阻塞队列。

怎么理解无界队列和有界队列？
- 有界队列即长度有限，满了以后ArrayBlockingQueue会插入阻塞。
- 无界队列就是里面能放无数的东西而不会因为队列长度限制被阻塞，但是可能会出现OOM异常。

线程池中的几种重要的参数及流程说明。
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
corePoolSize：核心池的大小，在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中

maximumPoolSize：线程池最大线程数最大线程数

keepAliveTime：表示线程没有任务执行时最多保持多久时间会终止

unit：参数keepAliveTime的时间单位TimeUtil类的枚举类（DAYS、HOURS、MINUTES、SECONDS 等）

workQueue：阻塞队列，用来存储等待执行的任务

threadFactory：线程工厂，主要用来创建线程

handler：拒绝处理任务的策略

AbortPolicy：丢弃任务并抛出 RejectedExecutionException 异常。（默认这种）

DiscardPolicy：也是丢弃任务，但是不抛出异常

DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）

CallerRunsPolicy：由调用线程处理该任务

#### 执行流程

当有任务进入时，线程池创建线程去执行任务，直到核心线程数满为止
核心线程数量满了之后，任务就会进入一个缓冲的任务队列中

当任务队列为无界队列时，任务就会一直放入缓冲的任务队列中，不会和最大线程数量进行比较
当任务队列为有界队列时，任务先放入缓冲的任务队列中，当任务队列满了之后，才会将任务放入线程池，此时会拿当前线程数与线程池允许的最大线程数进行比较，如果超出了，则默认会抛出异常。如果没超出，然后线程池才会创建线程并执行任务，当任务执行完，又会将缓冲队列中的任务放入线程池中，然后重复此操作。

## JVM相关
说一下对jvm的理解，jvm的组成部分，各个部分的存储内容以及常见的jvm的问题排查步骤。

对JVM熟不熟悉？简单说说类加载过程，里面执行的那些操作？

JVM方法区存储内容 是否会动态扩展 是否会出现内存溢出 出现的原因有哪些。

介绍介绍CMS。

介绍介绍G1。

为什么jdk8用metaspace数据结构用来替代perm？

简单谈谈堆外内存以及你的理解和认识。

JVM的内存模型的理解，threadlocal使用场景及注意事项？

JVM老年代和新生代的比例？

jstack,jmap,jutil分别的意义？如何线上排查JVM的相关问题？

Java虚拟机中，数据类型可以分为哪几类？

怎么理解栈、堆？堆中存什么？栈中存什么？

为什么要把堆和栈区分出来呢？栈中不是也可以存储数据吗？

在Java中，什么是是栈的起始点，同是也是程序的起始点？

为什么不把基本类型放堆中呢？

Java中的参数传递时传值呢？还是传引用？

Java中有没有指针的概念？

Java中，栈的大小通过什么参数来设置？

一个空Object对象的占多大空间？

对象引用类型分为哪几类？

讲一讲垃圾回收算法。

如何解决内存碎片的问题？

如何解决同时存在的对象创建和对象回收问题？

讲一讲内存分代及生命周期。

什么情况下触发垃圾回收？

如何选择合适的垃圾收集算法？

JVM中最大堆大小有没有限制？

堆大小通过什么参数设置？

JVM有哪三种垃圾回收器？

吞吐量优先选择什么垃圾回收器？响应时间优先呢？

如何进行JVM调优？有哪些方法？

如何理解内存泄漏问题？有哪些情况会导致内存泄露？如何解决？


- dubbo的底层负载均衡，容错机制都是怎么实现的？
https://www.cnblogs.com/juncaoit/p/7691411.html

- Dubbo实现了远端rpc调用，请手写一个rpc调用

- Redis为什么可以实现分布式锁，memcached可以实现分布式锁么?实现分布式锁的方式有很多种，为什么选择redis分布式锁?

- in-jvm(必考)以及jmm缓存模型如何调优?

- Concurrenthashmap为什么要用红黑树？为何不用其他的树，平衡二叉树，b+?
因为平衡二叉是高度平衡的树, 而每一次对树的修改, 都要 rebalance, 这里的开销会比红黑树大. 如果插入一个node引起了树的不平衡，平衡二叉树和红黑树都是最多只需要2次旋转操作，即两者都是O(1)；但是在删除node引起树的不平衡时，最坏情况下，平衡二叉树需要维护从被删node到root这条路径上所有node的平衡性，因此需要旋转的量级O(logN)，而红黑树最多只需3次旋转，只需要O(1)的复杂度, 所以平衡二叉树需要rebalance的频率会更高，因此红黑树在大量插入和删除的场景下效率更高。
http://www.importnew.com/23621.html

- 栈的特性先进后出。手写实现入栈出栈，获取栈的长度，栈是否为空。

- 一个树，从根节点往下走，每条路径的节点值为某一数值，不管最后节点是不是叶子节点。写出具体实现方法。

- 数据库的隔离级别
核心理解间隙锁的实现。

- 数据库是怎么搭建集群的，主从数据同步怎么做的?
数据库binlog

- 如何给hashmap的key对象设计他的hashcode?
巧妙的设计hash算法可以使的索引尽量分布均匀，提供查找效率。
http://blog.csdn.net/o9109003234/article/details/44107811

- 场景式的问题:秒杀,能列出常见的排队、验证码、库存扣减方式对系统高并发的影响?

## 参考
- https://mp.weixin.qq.com/s/hiIBQu6mAPzYa7QgTNacUA
- https://www.zhihu.com/question/27339390
- http://www.cnblogs.com/zuoxiaolong/p/life51.html