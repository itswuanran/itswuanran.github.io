---
title: JIT参考材料
date: 2019-03-25 16:07:29
tags:
- JIT
categories:
- Java
---

## JITWatch相关资料

> 官网：https://github.com/AdoptOpenJDK/jitwatch

> 第三方使用介绍：http://zeroturnaround.com/rebellabs/why-it-rocks-to-finally-understand-java-jit-with-jitwatch/

> 中文介绍：http://rongmayisheng.com/post/java-jit性能调优

> 优化实例：http://rongmayisheng.com/post/怎样让你的代码更好的被jvm-jit-inlining

> HotSpot生成汇编代码插件安装：https://www.chrisnewland.com/building-hsdis-amd64dylib-on-mac-osx-376

相关链接

很详细的两篇解释JIT各类优化策略的文章：

https://advancedweb.hu/2016/05/27/jvm_jit_optimization_techniques/

https://advancedweb.hu/2016/06/28/jvm_jit_optimization_techniques_part_2/

来自IBM的介绍: https://www.ibm.com/developerworks/cn/java/j-lo-just-in-time/

这个带视频: http://www.infoq.com/cn/articles/Java-Application-Hostile-to-JIT-Compilation

JIT Arch(推荐): https://www.infoq.com/articles/OpenJDK-HotSpot-What-the-JIT

JIT history: http://stackoverflow.com/questions/692136/when-did-java-get-a-jit-compiler

Wiki:

https://en.wikipedia.org/wiki/Java_performance

https://en.wikipedia.org/wiki/HotSpot

官方（OpenJDK）的JIT所有优化方法汇总:

https://wiki.openjdk.java.net/display/HotSpot/PerformanceTacticIndex

https://wiki.openjdk.java.net/display/HotSpot/PerformanceTechniques

HotSpot Dev Group: http://openjdk.java.net/groups/hotspot/

HotSpot中的名词解释: http://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html

Java7和8的performance enhancement

http://docs.oracle.com/javase/7/docs/technotes/guides/vm/performance-enhancements-7.html

https://docs.oracle.com/javase/8/docs/technotes/guides/vm/performance-enhancements-7.html

来自Oracle的JVM spec：http://docs.oracle.com/javase/specs/jvms/se8/html/index.html

很短的JIT讲解：https://docs.oracle.com/cd/E13150_01/jrockit_jvm/jrockit/geninfo/diagnos/underst_jit.html

美团的一篇关于乱序执行的文章：http://tech.meituan.com/java-memory-reordering.html

相关资料

来自IBM的一次会议报告，非常详细的解释了JIT做的工作: ACACES06.pdf

Intel早期的一篇论文（98年），可以看到对Java的JIT刚开始的设计: pldi98.ps

IBM的一个长文，简要解释了JIT技术，更重要的是提供了如何让代码更容易被JIT优化的guideline: IBM_just_in_time_compiler_for_java.pdf

JITWatch的介绍：UnderstandingHotSpotJVMPerformancewithJITWatch.pdf

全面介绍JVM的书，在国产书里算写得好的: 《深入理解Java虚拟机》（第二版）

Java性能优化的权威读物：《Java Performance: The Definitive Guide》