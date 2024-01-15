---
title: 反应式编程
date: 2020-11-24 14:34:11
tags:
- reactive
categories:
- reactive
---

## 反应式宣言
https://www.reactivemanifesto.org/

非反应式架构的问题（RPC/SOA）
1. 同步等待，大量线程block，load高 资源利用率低
2. 微服务化，在边界上划分开了，不能按照业务流程的依赖来并行，RT会累积
3. 为了隔离RT累积引入Cache，超时时间，实现复杂

最近对这么一句话很有感触：
“reactive这味药吃多了容易伤身，真的用了就会回过头来求coroutine来救命。”
业务模块使用协程, 全局编排reactive才是合理思路。而不是为了背压全部上reactive

所以协程是否会取代反应式？
随着jvm loom项目和kotlin语言协程的推出，一些人认为反应式没有必要了，因为通过开销更小的协程来进行IO相关请求，同样可以提高CPU的利用率，但只是用协程，无法做到资源的最大化利用，并且无背压等支持，以及在处理实时数据流等业务场景中较为困难。如果反应式中基于线程池的调度转为基于协程池进行调度，这样可以很容易将只支持同步的SQL等阻塞IO改造为符合反应式的风格异步IO

## 优秀专题
- https://www.jdon.com/reactive.html