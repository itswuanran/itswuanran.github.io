---
title: Java性能优化
date: 2018-03-26 15:18:58
categories:
- Java
tags:
- 性能
---
## 代码级别
- 使用JMH做微基准测试，不臆想，不猜测
- 更多优化
int vs Integer
4 byte vs 16 byte
Java对象最小16 bytes， 12 bytes的固定header，按8 bytes对齐
AtomicIntegerFieldUpdater vs AtomicInteger  
Netty大量使用此方法，代码非常复杂。 
海量对象，多个数字属性时:int + 静态的updater vs AtoimicInteger
ArrayList vs LinkedList 
Pointer-Based的容器，每个元素是Node对象，里面含真正对象及前后节点的指针(24 bytes per element)
Array-Based的容器，直接在数组里存放对象  (4 bytes per element)，而且是连续性存储的(缓存行预加载)
- StringBuilder扩容复制
http://calvin1978.blogcn.com/articles/stringbuilder.html

## 数据库
数据库的调优，总的来说分为以下三部分：

## SQL调优
慢查询日志，使用explain、profile等工具。

## 架构层面的调优
读写分离、多从库负载均衡、水平和垂直分库分表

## 连接池调优
我们的应用为了实现数据库连接的高效获取、对数据库连接的限流等目的，通常会采用连接池类的方案，即每一个应用节点都管理了一个到各个数据库的连接池。随着业务访问量或者数据量的增长，原有的连接池参数可能不能很好地满足需求，这个时候就需要结合当前使用连接池的原理、具体的连接池监控数据和当前的业务量作一个综合的判断，通过反复的几次调试得到最终的调优参数。

## 缓存
## 分类
本地缓存（HashMap/ConcurrentHashMap、Ehcache、Guava Cache等），缓存服务（Redis/Tair/Memcache等）。

## 使用场景
### 什么情况适合用缓存？考虑以下两种场景：
- 短时间内相同数据重复查询多次且数据更新不频繁，这个时候可以选择先从缓存查询，查询不到再从数据库加载并回设到缓存的方式。此种场景较适合用单机缓存。
- 高并发查询热点数据，后端数据库不堪重负，可以用缓存来扛。
### 选型考虑
- 如果数据量小，并且不会频繁地增长又清空（这会导致频繁地垃圾回收），那么可以选择本地缓存。具体的话，如果需要一些策略的支持（比如缓存满的逐出策略），可以考虑Ehcache；如不需要，可以考虑HashMap；如需要考虑多线程并发的场景，可以考虑ConcurentHashMap。
- 其他情况，可以考虑缓存服务。目前从资源的投入度、可运维性、是否能动态扩容以及配套设施来考虑，我们优先考虑Tair。除非目前Tair还不能支持的场合（比如分布式锁、Hash类型的value），我们考虑用Redis。
### 设计关键点
#### 什么时候更新缓存？如何保障更新的可靠性和实时性？
更新缓存的策略，需要具体问题具体分析。这里以门店POI的缓存数据为例，来说明一下缓存服务型的缓存更新策略是怎样的？目前约10万个POI数据采用了Tair作为缓存服务，具体更新的策略有两个：

- 接收门店变更的消息，准实时更新。
- 给每一个POI缓存数据设置5分钟的过期时间，过期后从DB加载再回设到DB。这个策略是对第一个策略的有力补充，解决了手动变更DB不发消息、接消息更新程序临时出错等问题导致的第一个策略失效的问题。通过这种双保险机制，有效地保证了POI缓存数据的可靠性和实时性。
#### 缓存是否会满，缓存满了怎么办？
对于一个缓存服务，理论上来说，随着缓存数据的日益增多，在容量有限的情况下，缓存肯定有一天会满的。如何应对？
① 给缓存服务，选择合适的缓存逐出算法，比如最常见的LRU。
② 针对当前设置的容量，设置适当的警戒值，比如10G的缓存，当缓存数据达到8G的时候，就开始发出报警，提前排查问题或者扩容。
③ 给一些没有必要长期保存的key，尽量设置过期时间。

#### 缓存是否允许丢失？丢失了怎么办？
根据业务场景判断，是否允许丢失。如果不允许，就需要带持久化功能的缓存服务来支持，比如Redis或者Tair。更细节的话，可以根据业务对丢失时间的容忍度，还可以选择更具体的持久化策略，比如Redis的RDB或者AOF。

#### 缓存被“击穿”问题
对于一些设置了过期时间的key，如果这些key可能会在某些时间点被超高并发地访问，是一种非常“热点”的数据。这个时候，需要考虑另外一个问题：缓存被“击穿”的问题。
- 概念：缓存在某个时间点过期的时候，恰好在这个时间点对这个Key有大量的并发请求过来，这些请求发现缓存过期一般都会从后端DB加载数据并回设到缓存，这个时候大并发的请求可能会瞬间把后端DB压垮。
- 如何解决：业界比较常用的做法，是使用mutex。简单地来说，就是在缓存失效的时候（判断拿出来的值为空），不是立即去load db，而是先使用缓存工具的某些带成功操作返回值的操作（比如Redis的SETNX或者Memcache的ADD）去set一个mutex key，当操作返回成功时，再进行load db的操作并回设缓存；否则，就重试整个get缓存的方法。类似下面的代码：
```java
  public String get(key) {
      String value = redis.get(key);
      if (value == null) { //代表缓存值过期
          //设置3min的超时，防止del操作失败的时候，下次缓存过期一直不能load db
          if (redis.setnx(key_mutex, 1, 3 * 60) == 1) {  //代表设置成功
               value = db.get(key);
                      redis.set(key, value, expire_secs);
                      redis.del(key_mutex);
              } else {  //这个时候代表同时候的其他线程已经load db并回设到缓存了，这时候重试获取缓存值即可
                      sleep(50);
                      get(key);  //重试
              }
          } else {
              return value;      
          }
  }
```

## 异步
## 异步线程池
guava eventbus
## 消息队列
kafka

## 并发与锁
## 锁优化
在并发程序中，对可伸缩性的最主要威胁就是独占方式的资源锁。
3种方式降低锁的竞争程度
* 减少锁的持有时间
synchronized 尽量短的代码块
但如果有多个断断续续的同步块，可考虑合并粗化
* 降低锁的持有频率。
ReadWriteLock 
多线程并发读锁，单线程写锁  
BlockingQueue 
ArrayBlockingQueue : 全局一把锁  
LinkedBlockingQueue : 队头队尾两把锁
* 使用带有协调机制的堵占锁，这些机制允许更高的并发性。
缩小锁的范围（同步代码块）
减小锁的粒度（）
锁分解，锁分段
ConcurrentHashMap 分段锁 16
避免热点域，降低竞争锁的影响 放弃使用独占锁，使用并发容器，读-写锁。不可变对象和原子变量。

## 无锁（lock-free）
- CAS 
* Compare And Set 自旋
* Atomic*系列
* ConcurrentLinkedQueue
- ThreadLocal
- 不可变对象
不修改对象属性  直接替换为新对象
CopyOnWriteArrayList

异步日志和无锁队列的实现
## 并发的其他话题
- ThreadLocal的代价 
没有银弹，开放地址法的HashMap 
Netty的 FastThreadLocal, index原子自增的数组 
- volatile的代价
Happen Before   
volatile vs Atomic*，可见性 vs 原子性 
访问开销比同步块小，但也不可忽视  存到临时变量使用，需要确保可见性时再次赋值
- 合理设置线程 内存，线程调度的消耗不可忽略
- ForkJoinPool的其他优势:每线程独立任务队列，无锁 Netty的EventLoop， 同样由一组容量为1的线程池组成
- Quasar 协程:少量线程处理大量并发请求
遇上阻塞，主动让出线程去做其他事情  阻塞返回，随便拉一条线程来再续前缘
Java中的纤程库 - Quasar: http://colobu.com/2016/07/14/Java-Fiber-Quasar/

## JVM调优
TODO

## 工具
- JITWatch
https://github.com/AdoptOpenJDK/jitwatch/
- Btrace
http://calvin1978.blogcn.com/articles/btrace1.html
- FastUtil,Koloboke
http://java-performance.info/hashmap-overview-jdk-fastutil-goldman-sachs-hppc-koloboke-trove-january-2015/

## 参考
- [Java性能优化指南V1.4]
- http://ifeve.com/disruptor/
- http://www.hollischuang.com/archives/1072
- https://tech.meituan.com/境外业务性能优化实践.html
- https://tech.meituan.com/performance_tunning.html
- http://www.hollischuang.com/archives/489


