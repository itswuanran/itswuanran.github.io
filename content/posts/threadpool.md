---
title: 线程池的策略
date: 2019-04-28 11:28:18
tags:
- think
categories:
- think
---

## 线程池提交策略

## 什么时候会新建线程
未到核心线程数时，直接新建
先提交到队列，再新建线程
```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        //workerCountOf(c)会获取当前正在运行的worker数量
        if (workerCountOf(c) < corePoolSize) {
            //如果workerCount小于corePoolSize,就创建一个worker然后直接执行该任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //isRunning(c)是判断线程池是否在运行中,如果线程池被关闭了就不会再接受任务
        //后面将任务加入到队列中
        if (isRunning(c) && workQueue.offer(command)) {
            //如果添加到队列成功了,会再检查一次线程池的状态
            int recheck = ctl.get();
            //如果线程池关闭了,就将刚才添加的任务从队列中移除
            if (!isRunning(recheck) && remove(command))
                //执行拒绝策略
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //如果加入队列失败,就尝试直接创建worker来执行任务
        else if (!addWorker(command, false))
            //如果创建worker失败,就执行拒绝策略
            reject(command);
}
```

## 使用直接提交策略，即SynchronousQueue
首先SynchronousQueue是无界的，也就是说他存数任务的能力是没有限制的，但是由于该Queue本身的特性，在某次添加元素后必须等待其他线程取走后才能继续添加。在这里不是核心线程便是新创建的线程，但是我们试想一样下，下面的场景。
```java
new ThreadPoolExecutor(  
            2, 3, 30, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(),
            new RecorderThreadFactory("CookieRecorderPool"),
            new ThreadPoolExecutor.CallerRunsPolicy());
```
当核心线程已经有2个正在运行. 
1. 此时继续来了一个任务（A），根据前面介绍的“如果运行的线程等于或多于 corePoolSize，则 Executor 始终首选将请求加入队列，而不添加新的线程。”,所以A被添加到queue中。 
2. 又来了一个任务（B），且核心2个线程还没有忙完，OK，接下来首先尝试1中描述，但是由于使用的SynchronousQueue，所以一定无法加入进去 
3. 此时便满足了上面提到的“如果无法将请求加入队列，则创建新的线程，除非创建此线程超出maximumPoolSize，在这种情况下，任务将被拒绝。”，所以必然会新建一个线程来运行这个任务。 
4. 暂时还可以，但是如果这三个任务都还没完成，连续来了两个任务，第一个添加入queue中，后一个呢？queue中无法插入，而线程数达到了maximumPoolSize，所以只好执行异常策略了。

所以在使用SynchronousQueue通常要求maximumPoolSize是无界的，这样就可以避免上述情况发生（如果希望限制就直接使用有界队列）。对于使用SynchronousQueue的作用jdk中写的很清楚：此策略可以避免在处理可能具有内部依赖性的请求集时出现锁。 
什么意思？如果你的任务A1，A2有内部关联，A1需要先运行，那么先提交A1，再提交A2，当使用SynchronousQueue我们可以保证，A1必定先被执行，在A1么有被执行前，A2不可能添加入queue中

## 注意点
Queue中的take，put操作是阻塞的，offer，poll操作为非阻塞的

## 迷惑点
- 为什么都说SynchronousQueue是无界的？这个可以认为是不存储元素的队列啊
- 此策略可以避免在处理可能具有内部依赖性的请求集时出现锁，这句话到底表示什么意思，举得例子感觉是不正确的

如果你的任务A1，A2有内部关联，A1需要先运行，那么先提交A1，再提交A2，当使用SynchronousQueue我们可以保证，A1必定先被执行，在A1么有被执行前，A2不可能添加入queue中
这个只保证了A2不能加入到queue中，但没法保证A2在A1之后执行啊。个人理解Cached线程池在offer队列失败时会尝试新建一个线程执行的

## 参考
- https://blog.csdn.net/u013332124/article/details/79587436（推荐）
- https://blog.csdn.net/a837199685/article/details/50619311
- http://ifeve.com/java-synchronousqueue/
- https://mp.weixin.qq.com/s/jISHo8-aKMPjjeCYGJILgg