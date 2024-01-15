---
title: 大Key多Key分拆方案
date: 2019-02-28 21:29:30
tags: 
- Redis
categorys:
- Redis
---

业务场景中经常会有各种大key多key的情况， 比如：
1： 单个简单的key存储的value很大
2： hash，set，zset，list中存储过多的元素（以万为单位）
3：一个集群存储了上亿的key，Key 本身过多也带来了更多的空间占用

由于redis是单线程运行的，如果一次操作的value很大会对整个redis的响应时间造成负面影响，所以，业务上能拆则拆，下面举几个典型的分拆方案。

## 单个简单的key存储的value很大
- 该对象需要每次都整存整取
可以尝试将对象分拆成几个key-value， 使用multiGet获取值，这样分拆的意义在于分拆单次操作的压力。
将操作压力平摊到多个redis实例中，降低对单个redis的IO影响。
- 该对象每次只需要存取部分数据
可以像第一种做法一样，分拆成几个key-value，也可以将这个存储在一个hash中，每个field代表一个具体的属性，使用hget,hmget来获取部分的value，使用hset，hmset来更新部分属性。
## hash， set，zset，list 中存储过多的元素

 类似于场景一种的第一个做法，可以将这些元素分拆。
以hash为例，原先的正常存取流程是  hget(hashKey, field); hset(hashKey, field, value)
现在，固定一个桶的数量，比如 10000， 每次存取的时候，先在本地计算field的hash值，模除 10000， 确定了该field落在哪个key上。

```
newHashKey = hashKey + (hash(field) % 10000);  
hset(newHashKey, field, value);
hget(newHashKey, field)
``` 
set, zset, list 也可以类似上述做法。但有些不适合的场景，比如，要保证 lpop 的数据的确是最早push到list中去的，这个就需要一些附加的属性，或者是在key的拼接上做一些工作（比如list按照时间来分拆）。

## 一个集群存储了上亿的key
如果key的个数过多会带来更多的内存空间占用，
- key本身的占用（每个key 都会有一个Category前缀）
- 集群模式中，服务端需要建立一些slot2key的映射关系，这其中的指针占用在key多的情况下也是浪费巨大空间
这两个方面在key个数上亿的时候消耗内存十分明显（Redis 3.2及以下版本均存在这个问题，4.0有优化）；

所以减少key的个数可以减少内存消耗，可以参考的方案是转Hash结构存储，即原先是直接使用Redis String的结构存储，现在将多个key存储在一个Hash结构中，具体场景参考如下：

一： key 本身就有很强的相关性，比如多个key 代表一个对象，每个key是对象的一个属性，这种可直接按照特定对象的特征来设置一个新Key——Hash结构， 原先的key则作为这个新Hash 的field。
举例说明： 原先存储的三个key，user.zhangsan-id = 123;   user.zhangsan-age = 18;  user.zhangsan-country = china;
这三个key本身就具有很强的相关特性，转成Hash存储就像这样
```
key =  user.zhangsan
field:id = 123;
field:age = 18;
field:country = china;
```
即redis中存储的是一个key ：user.zhangsan， 他有三个 field， 每个field + key 就对应原先的一个key。

二： key 本身没有相关性，预估一下总量，采取和上述第二种场景类似的方案，预分一个固定的桶数量
比如现在预估key的总数为 2亿，按照一个hash存储100个field来算，需要 2亿 / 100 = 200W 个桶 (200W 个key占用的空间很少，2亿可能有将近 20G )
原先比如有三个key：
```
user.123456789, user.987654321, user.678912345
```
现在按照200W 固定桶分就是先计算出桶的序号hash(123456789) % 200W，这里最好保证这个hash算法的值是个正数，否则需要调整下模除的规则；
这样算出三个key 的桶分别是1，2，2。所以存储的时候调用hset(key,  field, value)，读取的时候使用hget(key， field)
```
key1:  hset(user.1,123456789,value);
       hget(user.1,123456789);
key2:  hset(user.2,987654321,value);
       hget(user.2,987654321);
key3:  hset(user.2,678912345,value);
       hget(user.2,678912345);
```
注意两个地方：
1，hash取模对负数的处理；
2，预分桶的时候，一个hash中存储的值最好不要超过512，100左右较为合适

## 大Bitmap或布隆过滤器（Bloom）拆分
使用bitmap或布隆过滤器的场景，往往是数据量极大的情况，在这种情况下，Bitmap和布隆过滤器使用空间也比较大，比如用于userid匹配的布隆过滤器，就需要512MB的大小，这对redis来说是绝对的大value了。
这种场景下，我们就需要对其进行拆分，拆分为足够小的Bitmap，比如将512MB的大Bitmap拆分为1024个512KB的Bitmap。不过拆分的时候需要注意，要将每个key落在一个Bitmap上。有些业务只是把Bitmap拆开，但还是当做一个整体的bitmap看，所以一个key还是落在多个Bitmap上，这样就有可能导致一个key请求需要查询多个节点、多个Bitmap。被请求的值被hash到多个Bitmap上，也就是redis的多个key上，这些key还有可能在不同节点上，这样拆分显然大大降低了查询的效率。

因此我们所要做的是把所有拆分后的Bitmap当作独立的bitmap，然后通过hash将不同的key分配给不同的bitmap上，而不是把所有的小Bitmap当作一个整体。这样做后每次请求都只要取redis中一个key即可。

有同学可能会问，通过这样拆分后，相当于Bitmap变小了，会不会增加布隆过滤器的误判率？实际上是不会的，布隆过滤器的误判率是哈希函数个数k，集合元素个数n，以及Bitmap大小m所决定的，其约等于。因此如果我们在第一步，也就是在分配key给不同Bitmap时，能够尽可能均匀的拆分，那么n／m的值几乎是一样的，误判率也就不会改变。
同时，客户端也提供便利的api setBits/getBits用于一次操作同一个key的多个bit值 。

建议：k取13个，单个bloomfilter控制在512KB以下
