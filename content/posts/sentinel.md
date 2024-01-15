---
title: Sentinel
date: 2018-08-23 22:33:45
tags:
- "限流"
---
## Sentinel 是什么？

随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Sentinel 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。
Sentinel 具有以下特征:

- 丰富的应用场景： Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀，即突发流量控制在系统容量可以承受的范围；消息削峰填谷；实时熔断下游不可用应用，等等。
- 完备的监控功能： Sentinel 同时提供最实时的监控功能，您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- 简单易用的扩展点： Sentinel 提供简单易用的扩展点，您可以通过实现扩展点，快速的定制逻辑。例如定制规则管理，适配数据源等。
Sentinel 分为两个部分:

核心库（Java 客户端）不依赖任何框架/库，能够运行于所有 Java 运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。
控制台（Dashboard）基于 Spring Boot 开发，打包后可以直接运行，不需要额外的 Tomcat 等应用容器。

大家可能会问：Sentinel 和之前常用的熔断降级库 Netflix Hystrix 有什么异同呢？
https://github.com/alibaba/Sentinel/wiki/Sentinel-%E4%B8%8E-Hystrix-%E7%9A%84%E5%AF%B9%E6%AF%94


## 设计的亮点

### ClusterBuilderSlot数据统计
ClusterBuilderSlot 中为了统计 total statistics of the same resource in different context 使用了加锁的HashMap 而不是 ConcurrentHashMap，主要是因为程序运行时间越长这个Map越稳定，程序只会在最开始的时候用一次锁，而不用每次都持有锁
### 当前时间获取
在获取当前时间时使用一个TimeUtil工具类，daemon线程定时更新当前时间，因为对性能要求比较高，这个时间的获取也是有设计在里面的
### 指标数据统计粒度
指标数据统计时，设计之初就考虑了两种粒度，一种是秒级别，一种是分级别的
### 指标计数
指标计数时使用了LongAdder，有很多文章比较过LongAdder和AtomicLong差异了，这里简单说下，核心区别在与实现上，一个使用CAS，一个使用分段锁。
AtomicLong 为了实现正确的累加操作，如果并发量很大的话，使用CAS，cpu会花费大量的时间在试错上面，相当于一个spin的操作
LongAdder则是使用分段锁的思想来避免资源的竞争，最好的情况下，每个线程都有独立的计数器，这样可以大量减少并发操作
返回累加的和可能不是绝对准确的，因为调用这个方法时还有其他线程可能正在进行计数累加，sentinel的场景，计数操作并不是核心，这种情况使用LongAdder则是更优的一种考虑

### 滑动窗口更新
在旧的bucket落后当前时间时，MetricBucket需要进行位置交接时，reset操作和 clean-up操作不是一个原子操作，这时候有两种方案
#### sentinel 实现方式

分离了创建和更新操作，创建使用cas，更新使用可重入锁 updateLock.tryLock()，值得一提的是采用CAS来对时间框对象WindowWrap操作，避免使用一些效率低的关键字，这是在处理高并发情况下，对同一个对象操作的高效手法。
使用updateLock时，让一个线程独占锁资源去创建，没有获取到锁时使用yeild操作让出cpu时间片，主要是因为sentinel认为这个是很细小的场景，在大多数情况下是没有性能损耗的
```java

                /*
                 *   (old)
                 *             B0       B1      B2    NULL      B4
                 * |_______||_______|_______|_______|_______|_______||___
                 * ...    1200     1400    1600    1800    2000    2200  timestamp
                 *                              ^
                 *                           time=1676
                 *          startTime of Bucket 2: 400, deprecated, should be reset
                 *
                 * If the start timestamp of old bucket is behind provided time, that means
                 * the bucket is deprecated. We have to reset the bucket to current {@code windowStart}.
                 * Note that the reset and clean-up operations are hard to be atomic,
                 * so we need a update lock to guarantee the correctness of bucket update.
                 *
                 * The update lock is conditional (tiny scope) and will take effect only when
                 * bucket is deprecated, so in most cases it won't lead to performance loss.
                 */
```                 
#### hystrix的策略

若发现要创建新的bucket，同样使用锁，让一个线程去创建，别的线程则取出上一个bucket来进行处理（牺牲了一点时刻准确度，但换来了性能的提升）
```java

        /* if we didn't find the current bucket above, then we have to create one */
        /**
         * The following needs to be synchronized/locked even with a synchronized/thread-safe data structure such as LinkedBlockingDeque because
         * the logic involves multiple steps to check existence, create an object then insert the object. The 'check' or 'insertion' themselves
         * are thread-safe by themselves but not the aggregate algorithm, thus we put this entire block of logic inside synchronized.
         * 
         * I am using a tryLock if/then (http://download.oracle.com/javase/6/docs/api/java/util/concurrent/locks/Lock.html#tryLock())
         * so that a single thread will get the lock and as soon as one thread gets the lock all others will go the 'else' block
         * and just return the currentBucket until the newBucket is created. This should allow the throughput to be far higher
         * and only slow down 1 thread instead of blocking all of them in each cycle of creating a new bucket based on some testing
         * (and it makes sense that it should as well).
         * 
         * This means the timing won't be exact to the millisecond as to what data ends up in a bucket, but that's acceptable.
         * It's not critical to have exact precision to the millisecond, as long as it's rolling, if we can instead reduce the impact synchronization.
         * 
         * More importantly though it means that the 'if' block within the lock needs to be careful about what it changes that can still
         * be accessed concurrently in the 'else' block since we're not completely synchronizing access.
         * 
         * For example, we can't have a multi-step process to add a bucket, remove a bucket, then update the sum since the 'else' block of code
         * can retrieve the sum while this is all happening. The trade-off is that we don't maintain the rolling sum and let readers just iterate
         * bucket to calculate the sum themselves. This is an example of favoring write-performance instead of read-performance and how the tryLock
         * versus a synchronized block needs to be accommodated.
         */
```         
## 参考

- [Sentinel 与 Hystrix 的对比](https://developer.aliyun.com/article/623424)
- [Sentinel Wiki](https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D)
- [限流降级神器，带你解读阿里巴巴开源 Sentinel 实现原理](https://mp.weixin.qq.com/s/tmf-o2B5OhwWjbJeBNtckQ)