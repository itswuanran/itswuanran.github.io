---
title: 理解ReentrantLock
date: 2018-01-10 16:22:17
categories: Java
tags: [ReentrantLock,synchronized]
---

## 显式锁

协调共享对象的访问的三个关键词
synchronized 、volatile和lock

## Lock and ReentrantLock 
内置锁的局限性：

- 无法 中断一个正在等待获取锁的线程
- 无法 在请求获取一个锁时 无限等待下去
- 内置锁必须在获取该锁的代码块中释放，无法实现非阻塞结构的加锁规则


### 轮询锁和定时锁

同时获取两个锁时才进行操作
显式地索取和释放锁，那么在索取锁的时候可以设定一个超时时间，如果超过这个时间还没索取到锁，则不会继续堵塞而是放弃此次任务。
tryLock时设置
```java
public boolean trySendOnSharedLine(String message,  
                                   long timeout, TimeUnit unit)  
                                   throws InterruptedException {  
    long nanosToLock = unit.toNanos(timeout)  
                     - estimatedNanosToSend(message);  
    if (!lock.tryLock(nanosToLock, NANOSECONDS))  
        return false;  
    try {  
        return sendOnSharedLine(message);  
    } finally {  
        lock.unlock();  
    }  
} 
```

### 可中断的锁获取操作
lock.lockInterruptibly();
能够在获取锁的同时保持对中断的响应，tryLock同样也能响应中断，适用于实现 一个定时和可中断的锁获取
```java
    public boolean sendOnSharedLine(String message)
            throws InterruptedException {
        lock.lockInterruptibly();
        try {
            return cancellableSendOnSharedLine(message);
        } finally {
            lock.unlock();
        }
    }

    private boolean cancellableSendOnSharedLine(String message) throws InterruptedException {
        /* send something */
        return true;
    }
```

### 非块结构的加锁（非阻塞？）

- 内置锁是基于块结构的加锁
- 锁分段技术，不同的线程可以独立的对链表的不同部分进行操作。
每个节点的锁将保护链接指针以及该节点中存储的数据，遍历或者修改时，获取当前节点的锁，直到获得下个节点的锁。
连锁式加锁

## 公平性

非公平锁和公平锁，非公平锁允许插队。插队是指 当一个线程请求非公平锁时，在发出请求的同时该锁的状态变为可用，那么可跳过队列中所有等待线程。

## 如何选择synchronized和ReentrantLock
在一些内置锁无法满足的情况下，ReentrantLock可以作为一种高级工具：
实现 可定时、可轮询和可中断的锁获取操作，公平队列，非块结构的锁。
否则优先使用synchronized。

## 读-写锁
放宽粒度 允许读-读操作，多个线程同时读。
读数据时确保读取最新数据，读数据时不会又其他线程修改数据。
