---
title: GC算法
date: 2019-03-22 14:20:57
tags:
- JVM
categories:
- JVM
---
## Full GC为什么会stop the world
GC线程和用户线程存在资源竞争。

http://stackoverflow.com/questions/29846041/why-remark-phase-is-needed-on-concurrent-gc
https://blog.csdn.net/fei33423/article/details/70941939
cms 新生代使用的paral ser 回收. stop the world. ocation
cms 老生带分为 allocated 和 free.其中cms 垃圾回收器会将 allocated 分为 reachable 和 unreachable.
关键点: allocated时时在变
所以为什么remark要stop the world?:

用户线程和 c mark 线程存在并发竞争
- A 独立, B 引用 C.  cmark先遍历了 A, cmark 线程遍历到 B 时, 用户线程将 C 给了 A.  这时候 C 没有被 reachable 到. 最后认为 C 是 unreable 的.其实并不是
- 另外已分配的池也时时变大. 如果不锁住,也会被有问题.导致 unreachable 计算的有误.

https://segmentfault.com/q/1010000006692189
JVM有个叫做“安全点”和“安全区域”的东西，在发生GC时，所有的线程都会执行到“安全点”停下来。
在需要GC的时候，JVM会设置一个标志，当线程执行到安全点的时候会轮询检测这个标志，如果发现需要GC，则线程会自己挂起，直到GC结束才恢复运行。

还有另一种策略是在GC发生时，直接把所有线程都挂起，然后检测所有线程是否都在安全点，如果不在安全点则恢复线程的执行，等执行到安全点再挂起。

但是对于一些没有获得或无法获得CPU时间的线程，就没办法等到它执行到安全点了，所以这个时候只要这个线程是在安全区域的，也可以进行GC，安全区域是一段代码段，在这段代码段中对象的引用关系不会发生变化，所以这个时候进行GC也是安全的。


## 可达性分析
- 引用计数法
即每个对象如果被引用，则对引用的次数+1，当引用为0时，则表示该对象不可达。
该方法无法解决互象引用的问题，从而导致内存泄漏。
- 可达性分析，即根据GC Roots 是否可达，来决定对象是否存活；
GC roots 对象包括：JVM方法栈中的对象及临时变量；本地方法栈中引用的对象；进程中的线程对象；静态资源

## 标记-清除算法
简单明了，把需要回收的对象标记，然后直接清理。
（1）、第一次标记
在可达性分析后发现对象到GC Roots没有任何引用链相连时，被第一次标记；
并且进行一次筛选：此对象是否必要执行finalize()方法；
对有必要执行finalize()方法的对象，被放入F-Queue队列中；    

（2）、第二次标记
GC将对F-Queue队列中的对象进行第二次小规模标记；
在其finalize()方法中重新与引用链上任何一个对象建立关联，第二次标记时会将其移出"即将回收"的集合；
对第一次被标记，且第二次还被标记（如果需要，但没有移出"即将回收"的集合），就可以认为对象已死，可以进行回收

- 标记和清除的效率都不高
标记压缩算法，两种常用的压缩算法：https://blog.csdn.net/jj_nan/article/details/72614701
two-Finger 算法相对简单，但只能处理固定大小的对象
lisp2可以处理任意大小，但相对复杂的寻址（标记地址和地址迁移）都要进行一系统复杂计算。
标记压缩算法的复杂性在于寻址算法的复杂性（即将可达对象存放在地址的某处的计算）

- 会出现大量不连续的内存碎片，碎片太多会导致无法分配大对象提前触发另外一次GC

## 复制算法
内存分为大小相同的两块，每次只使用一块，回收时，将存活的对象复制到另外一块区域，然后再把已使用过的内存空间一次性清理。
这种思路适合新生代的收集，新生代中的对象98%是朝生夕死。Eden和两块较小的Survivor空间，默认8:1:1。只是10%的空间会浪费。
复制算法解决了寻址的功能，通过简单计算偏移量即可解决寻址的问题，算法相对简单，但其要始终浪费一块空间来进行ping-pong存储。所以其缺陷在于内存利用率低。

## 标记整理
适合用于老年代的算法（存活对象多于垃圾对象）。标记后不复制，而是将存活对象压缩到内存的一端，然后清理边界外的所有对象。


## GC的一些区别
https://www.zhihu.com/question/41922036/answer/93079526

针对HotSpot VM的实现，它里面的GC其实准确分类只有两大种：
## Partial GC：并不收集整个GC堆的模式
- Young GC：只收集young gen的GC
- Old GC：只收集old gen的GC。只有CMS的concurrent collection是这个模式
- Mixed GC：收集整个young gen以及部分old gen的GC。只有G1有这个模式
## Full GC：收集整个堆，包括young gen、old gen、perm gen（如果存在的话）等所有部分的模式。
Major GC通常是跟full GC是等价的，收集整个GC堆。但因为HotSpot VM发展了这么多年，外界对各种名词的解读已经完全混乱了，当有人说“major GC”的时候一定要问清楚他想要指的是上面的full GC还是old GC。

最简单的分代式GC策略，按HotSpot VM的serial GC的实现来看，触发条件是：
- young GC：当young gen中的eden区分配满的时候触发。注意young GC中有部分存活对象会晋升到old gen，所以young GC后old gen的占用量通常会有所升高。
- full GC：当准备要触发一次young GC时，如果发现统计数据说之前young GC的平均晋升大小比目前old gen剩余的空间大，则不会触发young GC而是转为触发full GC（因为HotSpot VM的GC里，除了CMS的concurrent collection之外，其它能收集old gen的GC都会同时收集整个GC堆，包括young gen，所以不需要事先触发一次单独的young GC）；或者，如果有perm gen的话，要在perm gen分配空间但已经没有足够空间时，也要触发一次full GC；或者System.gc()、heap dump带GC，默认也是触发full GC。

HotSpot VM里其它非并发GC的触发条件复杂一些，不过大致的原理与上面说的其实一样。当然也总有例外。Parallel Scavenge（-XX:+UseParallelGC）框架下，默认是在要触发full GC前先执行一次young GC，并且两次GC之间能让应用程序稍微运行一小下，以期降低full GC的暂停时间（因为young GC会尽量清理了young gen的死对象，减少了full GC的工作量）。控制这个行为的VM参数是-XX:+ScavengeBeforeFullGC。这是HotSpot VM里的奇葩嗯。
https://www.zhihu.com/question/48780091/answer/113063216
并发GC的触发条件就不太一样。以CMS GC为例，它主要是定时去检查old gen的使用量，当使用量超过了触发比例就会启动一次CMS GC，对old gen做并发收集。

## CMS
CMS收集的方法是：先3次标记，再1次清除，3次标记中前两次是初始标记和重新标记（此时仍然需要停止（stop the world））， 初始标记（Initial Remark）是标记GC Roots能关联到的对象（即有引用的对象），停顿时间很短；并发标记（Concurrent remark）是执行GC Roots查找引用的过程，不需要用户线程停顿；重新标记（Remark）是在初始标记和并发标记期间，有标记变动的那部分仍需要标记，所以加上这一部分 标记的过程，停顿时间比并发标记小得多，但比初始标记稍长。在完成标记之后，就开始并发清除，不需要用户线程停顿。
所以在CMS清理过程中，只有初始标记和重新标记需要短暂停顿，并发标记和并发清除都不需要暂停用户线程，因此效率很高，很适合高交互的场合。
CMS也有缺点，它需要消耗额外的CPU和内存资源，在CPU和内存资源紧张，CPU较少时，会加重系统负担（CMS默认启动线程数为(CPU数量+3)/4）。
另外，在并发收集过程中，用户线程仍然在运行，仍然产生内存垃圾，所以可能产生“浮动垃圾”，本次无法清理，只能下一次Full GC才清理，因此在GC期间，需要预留足够的内存给用户线程使用。所以使用CMS的收集器并不是老年代满了才触发Full GC，而是在使用了一大半（默认68%，即2/3，使用-XX:CMSInitiatingOccupancyFraction来设置）的时候就要进行Full GC，如果用户线程消耗内存不是特别大，可以适当调高-XX:CMSInitiatingOccupancyFraction以降低GC次数，提高性能，如果预留的用户线程内存不够，则会触发Concurrent Mode Failure，此时，将触发备用方案：使用Serial Old 收集器进行收集，但这样停顿时间就长了，因此-XX:CMSInitiatingOccupancyFraction不宜设的过大。
还有，CMS采用的是标记清除算法，会导致内存碎片的产生，可以使用-XX：+UseCMSCompactAtFullCollection来设置是否在Full GC之后进行碎片整理，用-XX：CMSFullGCsBeforeCompaction来设置在执行多少次不压缩的Full GC之后，来一次带压缩的Full GC。

## 参考
- https://www.zhihu.com/question/41922036/answer/93079526
- http://www.importnew.com/1993.html
- http://www.importnew.com/2057.html
