---
title: LockFree
date: 2018-03-23 19:44:29
tags: 
- LockFree
---

很早看过一篇文章，介绍无锁缓存
[如何实现超高并发的无锁缓存？](http://mp.weixin.qq.com/s/bu86p2IcOFmHW5TJ9Xc_2A)
在【超高并发】，【写多读少】，【定长value】的【业务缓存】场景下：思路如下
1. 可以通过水平拆分来降低锁冲突
2. 可以通过Map转Array的方式来最小化锁冲突，一条记录一个锁
3. 可以把锁去掉，最大化并发，但带来的数据完整性的破坏
4. 可以通过签名的方式保证数据的完整性，实现无锁缓存


## Akka actor
实现无锁还有一种思路是actor mailbox，通过partition key将消息路由到不同的消费方，将有竞性条件的数据通过排队来处理

## LockFree Queue
Java 语言实现
- https://github.com/LMAX-Exchange/disruptor

Go 语言实现
- https://github.com/bruceshao/lockfree


## See Also
- [LockFree Pdf](https://www.cs.cmu.edu/~410-s05/lectures/L31_LockFree.pdf)
