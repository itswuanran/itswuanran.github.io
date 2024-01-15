---
title: ThreadLocal内存泄露
date: 2018-01-24 19:16:43
tags: [ThreadLocal]
categories:
- Java
---
## ThreadLocal作用
```
/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * <tt>get</tt> or <tt>set</tt> method) has its own, independently initialized
 * copy of the variable.  <tt>ThreadLocal</tt> instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 *
 * <p>For example, the class below generates unique identifiers local to each
 * thread.
 * A thread's id is assigned the first time it invokes <tt>ThreadId.get()</tt>
 * and remains unchanged on subsequent calls.
 * <pre>
 * import java.util.concurrent.atomic.AtomicInteger;
 *
 * public class ThreadId {
 *     // Atomic integer containing the next thread ID to be assigned
 *     private static final AtomicInteger nextId = new AtomicInteger(0);
 *
 *     // Thread local variable containing each thread's ID
 *     private static final ThreadLocal&lt;Integer> threadId =
 *         new ThreadLocal&lt;Integer>() {
 *             &#64;Override protected Integer initialValue() {
 *                 return nextId.getAndIncrement();
 *         }
 *     };
 *
 *     // Returns the current thread's unique ID, assigning it if necessary
 *     public static int get() {
 *         return threadId.get();
 *     }
 * }
 * </pre>
 * <p>Each thread holds an implicit reference to its copy of a thread-local
 * variable as long as the thread is alive and the <tt>ThreadLocal</tt>
 * instance is accessible; after a thread goes away, all of its copies of
 * thread-local instances are subject to garbage collection (unless other
 * references to these copies exist).
 *
 * @author  Josh Bloch and Doug Lea
 * @since   1.2
 */
```

ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。

## ThreadLocal的实现
ThreadLocal的实现是这样的：
每个Thread 维护一个ThreadLocalMap映射表，这个映射表的key是ThreadLocal实例本身，value是真正需要存储的对象Object。

也就是说ThreadLocal本身并不存储值，它只是作为一个key来让线程从ThreadLocalMap获取value。
但ThreadLocalMap是使用ThreadLocal的弱引用作为Key的，弱引用的对象在GC时会被回收。

## ThreadLocal为什么会内存泄漏
ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用来引用它，那么系统 GC 的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value永远无法回收，造成内存泄漏。

这个强引用链个人理解是指线程对ThreadLocalMap的，会始终存在一个null值为key的value，这个value一直存储在ThreadLocalMap中，除非线程终止，或者手动清除map中的null key，否则无法回收，多个线程都有这样的问题话，可能会导致内存泄漏。

## 参考
- http://www.importnew.com/22046.html
- https://mp.weixin.qq.com/s/VeL9tMavp4ppv3j2w9hwVg
- https://www.zhihu.com/question/23089780
- https://mp.weixin.qq.com/s?__biz=MzA3MDExNzcyNA==&mid=2650392118&idx=1&sn=a2144a19bdeba48001f4f76f423e25d9&scene=0#wechat_redirect