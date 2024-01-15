---
title: kotlin入门
date: 2020-11-23 01:58:57
tags:
- kotlin
categories:
- kotlin
---

## 对suspend的理解

suspend 标记只是一个提醒，作用在方法上只是表明“该方法会比较耗时，建议在协程中运行”
如果要实现程序的运行，需要将方法包装在这个结构中
```
CoroutineScope(Dispatchers.Default).launch{ }
CoroutineScope(Dispatchers.Default).async{ }
```
async/await做为库函数提供了
async的返回结果是个Deferred，类似CompletableFuture
kotlin提供了一个库可以将两者互转

## 详细资料

https://kaixue.io/kotlin-coroutines-1/
https://kaixue.io/kotlin-coroutines-2/
https://kaixue.io/kotlin-coroutines-3/


## kotlin coroutine的实现

CPS，stackless

