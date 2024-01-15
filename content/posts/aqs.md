---
title: 深入理解AQS相关问题
date: 2019-02-25 14:41:26
tags:
- Concurrent
categories:
- Java
---

## AQS为什么采用双向队列，只用单向会有什么问题？

多线程下唤醒线程时，存在从后往前遍历的情况。非公平锁，唤醒方向不确定

unparkSuccessor 里面的确有从尾部向前遍历的代码，但是为什么要这么做？
进入队列需要找到安全的休息点，需要访问前驱节点，前驱节点可能挂掉，新来的不能一直等着它后面，不然无法被唤醒

前置节点有一个比较重要的原因：等待队列的线程判断当前线程是否抢占成功，还是阻塞。
取消节点的next指向了自己。
因此unparkSuccessor的时候，如果node已经被取消，则无法通过next获取下一个node，新来的同步请求会放到队列尾部，这时将新节点的pre指针指向原tail，然后cas，最后设置原tail的next，这种情况下在设置next之前，s是可能为空的，可查看enq方法
当前node在获取锁成功时，会修改前驱head节点的next=null。
此时前驱head节点对应的线程，如果调用unparkSuccessor(head)，则会遇到head.next=null的情况
因此node的next指针在取消时：会指向自己（node.next=node) 在其后继节点，获取锁时：会置为null，因此在某些情况，需要通过prev指针遍历队列

```java
try{
    boolean interrupted = false;//判断自旋过程中是否被中断过
    for(;;){
        final Node p = node.predecessor();//获取前继节点
            if(p == head && tryAcquire(arg)){//如果当前的这个节点的前继节点是头节点就去尝试获取了同步状态
                setHead(node);//设为头节点
```

## ConcurrentHashMap 中对key进行加锁，防止对value多次实例化

- synchronizing on interned strings

```
I'd second that NON-recommendation of synchronizing on interned strings.  If you're interested in a high-performance in-memory cache with good concurrency, I would recommend checking out Ehcache.  In particular, I would also look at the SelfPopulatingCache wrapper that can provide this kind of cache loading behavior (and safely handle common cases like the lazy load below).


String.intern is fine for interning a String, but then, as others have pointed out, the questions remain:

1. What happens when several threads lookup the same key at about the same time? (dogpile)
2. What else is using these unique Strings as locks? (public lock)
3. What happens if/when a value can't be generated? (failure)
4. What happens when cache entries are no longer used? (garbage)

```

- 通过代理对象，思路不错，重复创建，延迟实例化，实例化的时候可以加锁了。jdk8的话有computeIfAbsent，会block，不适用于创建时间过久的对象
  
```
     * If the specified key is not already associated with a value,
     * attempts to compute its value using the given mapping function
     * and enters it into this map unless {@code null}.  The entire
     * method invocation is performed atomically, so the function is
     * applied at most once per key.  Some attempted update operations
     * on this map by other threads may be blocked while computation
     * is in progress, so the computation should be short and simple,
     * and must not attempt to update any other mappings of this map.
```

- putIfAbsent，先放一个占位的 Object，占位成功再替换为 费时间的 真正的value，这样value只会初始化一次。 缺点时，读取对象时要加一次判断，可能要用到强制类型转换
- 也可以参考java并发编程实战里的关于缓存的例子 Memoizer

```java
public class Memoizer<A, V> implements Computable<A, V> {
    private final ConcurrentMap<A, Future<V>> cache
            = new ConcurrentHashMap<A, Future<V>>();
    private final Computable<A, V> c;

    public Memoizer(Computable<A, V> c) {
        this.c = c;
    }

    public V compute(final A arg) throws InterruptedException {
        while (true) {
            Future<V> f = cache.get(arg);
            if (f == null) {
                Callable<V> eval = new Callable<V>() {
                    public V call() throws InterruptedException {
                        return c.compute(arg);
                    }
                };
                FutureTask<V> ft = new FutureTask<V>(eval);
                f = cache.putIfAbsent(arg, ft);
                if (f == null) {
                    f = ft;
                    ft.run();
                }
            }
            try {
                return f.get();
            } catch (CancellationException e) {
                cache.remove(arg, f);
            } catch (ExecutionException e) {
                throw LaunderThrowable.launderThrowable(e.getCause());
            }
        }
    }
}
```