---
title: 理解coroutine
date: 2018-01-22 13:43:18
tags:
- coroutine
categories:
- 语言特性
---

## 什么是coroutine？

> 近几年在高性能web server上node.js、异步IO等概念火了起来，协程作为异步IO的最佳封装方式又被提及起来。假设为了减少线程数量和切换开销，我们的web server运行在单个线程，依次处理请求，某个请求的处理中调用了阻塞函数会影响后续请求，但我们可以使用异步API:

> 假设想让业务中的多个RPC可以并行，通过线程池创建Future来实现的代价会很大。
有一个应用的请求处理被拆成了一堆可并发的模块，默认情况下顺序方法调用这些模块，而开启选项是可以使用线程池来并发调用这些模块达到降低RT的目的。该应用设计时未考虑有的模块中是不用进行RPC的，但为了确保需要RPC的部分能正确并发，仍然要通过线程池调用。处理一个请求就需要200次的线程池操作，因此当开启该选项时系统load、cpu都明显吃紧，很快进入限流。

将线程实现切换到协程后(创建多个协程在当前线程内部调度)，我们获得了RT降低的效果，且cpu 使用率，load的情况可以和顺序调用媲美，因为一个普通方法调用和使用协程去执行一个方法然后切换回来(没有任何阻塞的情况)相比之多了几十条指令，那些没有阻塞的模块用异步方式去调用并没有带来很多损耗

- https://en.wikipedia.org/wiki/Coroutine#Comparison_with_subroutines

协程是自己程序调度的，逻辑上并行执行，底层上非并行执行
线程是操作系统调度的，逻辑和底层上都是并行执行

> 用户控制切换，自动保存上下文状态，切换之间可以通过参数通信，可用同步的方式实现异步。这是我理解中的协程最重要的几个特征。

协程的切换开销远远小于内核对于线程的切换开销，因此对于IO密集型的程序，使用协程的优势在于用户可以像写同步IO一样，无需关心IO异步接口的细节。封装等待IO返回的逻辑，跳转到coroutine库进行调度。减小使用多个线程做同步IO带来的内核大量线程切换的开销。

## coroutine和goroutine对比

- https://stackoverflow.com/questions/18058164/is-a-go-goroutine-a-coroutine

## 协程在各语言中的实现应用

## kotlin

- http://www.kotlincn.net/docs/reference/coroutines/coroutines-guide.html
- https://vertx.io/docs/vertx-lang-kotlin-coroutines/kotlin/

## php

- http://nikic.github.io/2012/12/22/Cooperative-multitasking-using-coroutines-in-PHP.html

## lua

- http://www.lua.org/pil/9.1.html
- https://segmentfault.com/a/1190000004207662

## python

- http://www.dabeaz.com/coroutines/Coroutines.pdf

## node.js

> 首先coroutine是个很宽泛的概念，async/await也属于coroutine的一种。但是问题是拿async/await和stackful coroutine比较。所谓stackful是指每个coroutine有独立的运行栈，比如每个goroutine会分配一个4k的内存来做为运行栈，切换goroutine的时候运行栈也会切换。stackful的好处在于这种coroutine是完整的，coroutine可以嵌套、循环。与stackful对应的是stackless coroutine，比如generator,continuation，这类coroutine不需要分配单独的栈空间，coroutine状态保存在闭包里，但缺点是功能比较弱，不能被嵌套调用，也没办法和异步函数配合使用进行控制流的调度，所以基本上没办法跟stackful coroutine做比较。但是async/await的出现，实现了基于stackless coroutine的完整coroutine。在特性上已经非常接近stackful coroutine了，不但可以嵌套使用也可以支持try catch。所以是不是可以认为async/await是一个更好的方案？

- http://es6.ruanyifeng.com/#docs/generator-async
- http://es6.ruanyifeng.com/#docs/promise
- http://blog.csdn.net/shenlei19911210/article/details/61194617
- http://blog.csdn.net/lilongsy/article/details/74333802
- https://www.zhihu.com/question/65647171/answer/233781797

## 扩展-并发

- http://jolestar.com/parallel-programming-model-thread-goroutine-actor/
- http://www.infoq.com/cn/presentations/free-performance-lunch-alibaba-jdk-association