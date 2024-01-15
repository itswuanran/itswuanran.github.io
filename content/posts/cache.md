---
title: "热点缓存识别方案"
date: 2023-06-15T17:35:40+08:00
draft: false
---

- [Tair黑魔法之HotRing](https://www.cnblogs.com/lizhaolong/p/16437181.html)

- [一碰就头疼的缓存热点Key问题，阿里的Tair是如何解决的？](https://blog.51cto.com/u_15009384/2563892)

- [有赞透明多级缓存解决方案（TMC）](https://mp.weixin.qq.com/s?__biz=MzAxOTY5MDMxNA==&mid=2455759090&idx=1&sn=f9f0b49d7c1916672f9d4f63dab0c2b6&chksm=8c686ed7bb1fe7c1446838941ff1bdb5d0bd8738aa43c22d456cf9736e3068eb13a29f908403&scene=21#wechat_redirect)

- [京东毫秒级热 key 探测框架设计与实践，已实战于 618 大促](https://my.oschina.net/1Gk2fdm43/blog/4331985)

Tair的解决方案

题外话：Tair也有解决单个节点性能瓶颈的能力。假设双十一淘宝首页上的一件商品，我们可知，这件商品信息的TPS起码是几百万甚至千万级别的。这样的热点KEY，无论你的集群有多大，这个KEY只会路由到其中的一台服务器上，那么并发能力就受到单节点限制，原生的memcache和Redis是完全搞不定的（Redis单节点TPS大概是10w级别，memcache可以达到几十万）。有一种办法就是自己处理本地缓存，这样代价比较大。阿里Tair的做法是热点散列，如下图所示，在每一个DataServer上开辟一个HotZone，统计到的热点KEY就保存到集群每一个节点上的HotZone中，客户端把热点数据KEY的请求随机打到任意一台DataServer的HotZone区域（热点KEY并不是Hash取模，这一点很重要），这样的话热点KEY请求就会被散列到多个节点乃至整个集群。那么整个Tair集群就不会因为某个热点KEY而发生过载的情况。本文不作过多的发散，有兴趣的同学请戳链接了解更多，非常值得一看：2017双11技术揭秘—分布式缓存服务Tair的热点数据散列机制：https://yq.aliyun.com/articles/316466。