---
title: thread-local-transmit
date: 2018-06-11 17:57:15
tags: 
- ThreadLocal
categories:
- Java
---
## 前言
介绍InheritableThreadLocal之前，假设对 ThreadLocal 已经有了一定的理解，比如基本概念，原理，如果没有，
可以参考：[ThreadLocal源码分析解密](https://blog.csdn.net/a837199685/article/details/50806876)
在讲解之前我们先列举有关ThreadLocal的几个关键点

每一个Thread线程都有属于自己的ThreadLocalMap,里面有一个弱引用的Entry(ThreadLocal,Object),如下
```java
Entry(ThreadLocal k, Object v) {
 super(k);
 value = v;
}
```

从ThreadLocal中get值的时候,首先通过Thread.currentThread得到当前线程，然后拿到这个线程的ThreadLocalMap。再传递当前ThreadLocal对象（结合上一点）。取得Entry中的value值

set值的时候同理，更改的是当先线程的ThreadLocalMap中Entry中key为当前Threadlocal对象的value值
## Threadlocal bug？
如果子线程想要拿到父线程的中的ThreadLocal值怎么办呢？比如会有以下的这种代码的实现。由于ThreadLocal的实现机制,在子线程中get时,我们拿到的Thread对象是当前子线程对象,那么他的ThreadLocalMap是null的,所以我们得到的value也是null。
```java
final ThreadLocal threadLocal=new ThreadLocal(){
 @Override
 protected Object initialValue() {
 return "xiezhaodong";
 }
 };
 new Thread(new Runnable() {
 @Override
 public void run() {
 threadLocal.get();//NULL
 }
 }).start();
```

## InheritableThreadLocal实现
那其实很多时候我们是有子线程获得父线程ThreadLocal的需求的,要如何解决这个问题呢？这就是InheritableThreadLocal这个类所做的事情。先来看下InheritableThreadLocal所做的事情。
```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
 
 protected T childValue(T parentValue) {
 return parentValue;
 }
 
 /**  * 重写Threadlocal类中的getMap方法，在原Threadlocal中是返回  *t.theadLocals，而在这么却是返回了inheritableThreadLocals，因为  * Thread类中也有一个要保存父子传递的变量  */
 ThreadLocalMap getMap(Thread t) {
 return t.inheritableThreadLocals;
 }
 
 /**  * 同理，在创建ThreadLocalMap的时候不是给t.threadlocal赋值  *而是给inheritableThreadLocals变量赋值  *   */
 void createMap(Thread t, T firstValue) {
 t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
 }
```

以上代码大致的意思就是，如果你使用InheritableThreadLocal,那么保存的所有东西都已经不在原来的t.thradLocals里面，而是在一个新的t.inheritableThreadLocals变量中了。下面是Thread类中两个变量的定义
```java
/* ThreadLocal values pertaining to this thread. This map is maintained  * by the ThreadLocal class. */
 ThreadLocal.ThreadLocalMap threadLocals = null;
 
 /*  * InheritableThreadLocal values pertaining to this thread. This map is  * maintained by the InheritableThreadLocal class.  */
 ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

Q：InheritableThreadLocal是如何实现在子线程中能拿到当前父线程中的值的呢？
A：一个常见的想法就是把父线程的所有的值都copy到子线程中。
下面来看看在线程new Thread的时候线程都做了些什么？
```java
private void init(ThreadGroup g, Runnable target, String name,  long stackSize, AccessControlContext acc) {
 //省略上面部分代码
 if (parent.inheritableThreadLocals != null)
 //这句话的意思大致不就是，copy父线程parent的map，创建一个新的map赋值给当前线程的inheritableThreadLocals。
 this.inheritableThreadLocals =
 ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
 //ignore
 }
```

而且，在copy过程中是浅拷贝，key和value都是原来的引用地址
```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
 Entry[] parentTable = parentMap.table;
 int len = parentTable.length;
 setThreshold(len);
 table = new Entry[len];
 for (int j = 0; j < len; j++) {
 Entry e = parentTable[j];
 if (e != null) {
 ThreadLocal key = e.get();
 if (key != null) {
 Object value = key.childValue(e.value);
 Entry c = new Entry(key, value);
 int h = key.threadLocalHashCode & (len - 1);
 while (table[h] != null)
 h = nextIndex(h, len);
 table[h] = c;
 size++;
 }
 }
 }
 }
```

恩，到了这里，大致的解释了一下InheritableThreadLocal为什么能解决父子线程传递Threadlcoal值的问题。
在创建InheritableThreadLocal对象的时候赋值给线程的t.inheritableThreadLocals变量
在创建新线程的时候会check父线程中t.inheritableThreadLocals变量是否为null，如果不为null则copy一份ThradLocalMap到子线程的t.inheritableThreadLocals成员变量中去
因为复写了getMap(Thread)和CreateMap()方法,所以get值得时候，就可以在getMap(t)的时候就会从t.inheritableThreadLocals中拿到map对象，从而实现了可以拿到父线程ThreadLocal中的值
so,在最开始的代码示例中，如果把ThreadLocal对象换成InheritableThreadLocal对象，那么get到的字符会是“xiezhaodong”而不是NULL

## InheritableThreadLocal还有问题吗？
问题场景
我们在使用线程的时候往往不会只是简单的new Thrad对象，而是使用线程池，当然线程池的好处多多。这里不详解，既然这里提出了问题，那么线程池会给InheritableThreadLocal带来什么问题呢？我们列举一下线程池的特点：

- 为了减小创建线程的开销，线程池会缓存已经使用过的线程
- 生命周期统一管理,合理的分配系统资源

对于第一点，如果一个子线程已经使用过，并且会set新的值到ThreadLocal中，那么第二个task提交进来的时候还能获得父线程中的值吗？比如下面这种情况(虽然是线程，用sleep尽量让他们串行的执行):

```java
final InheritableThreadLocal<Span> inheritableThreadLocal = new InheritableThreadLocal<Span>();
 inheritableThreadLocal.set(new Span("xiexiexie"));
 //输出 xiexiexie
 Object o = inheritableThreadLocal.get();
 Runnable runnable = new Runnable() {
 @Override
 public void run() {
 System.out.println("========");
 inheritableThreadLocal.get();
 inheritableThreadLocal.set(new Span("zhangzhangzhang");
 inheritableThreadLocal.get();
 }
 };
 
 ExecutorService executorService = Executors.newFixedThreadPool(1);
 executorService.submit(runnable);
 TimeUnit.SECONDS.sleep(1);
 executorService.submit(runnable);
 TimeUnit.SECONDS.sleep(1);
 System.out.println("========");
 Span span = inheritableThreadLocal.get();
 }
 static class Span {
 public String name;
 public int age;
 public Span(String name) {
 this.name = name;
 }
 }
```

输出的会是
```
xiexiexie
 ========
xiexiexie
zhangzhangzhang
 ========
zhangzhangzhang
zhangzhangzhang
 ========
xiexiexie
```
造成这个问题的原因是什么呢，下图大致讲解一下整个过程的变化情况，如图所示，由于B任务提交的时候使用了，A任务的缓存线程，A缓存线程的InheritableThreadLocal中的value已经被更新成了”zhangzhangzhang“。B任务在代码内获得值的时候，直接从t.InheritableThreadLocal中获得值，所以就获得了线程A中心设置的值，而不是父线程中InheritableThreadLocal的值。

so，InheritableThreadLocal还是不能够解决线程池当中获得父线程中ThreadLocal中的值。

## 造成问题的原因
那么造成这个问题的原因是什么呢？如何让任务之间使用缓存的线程不受影响呢？实际原因是，我们的线程在执行完毕的时候并没有清除ThreadLocal中的值，导致后面的任务重用现在的localMap。

## 解决方案
如果我们能够，在使用完这个线程的时候清除所有的localMap，在submit新任务的时候在重新重父线程中copy所有的Entry。然后重新给当前线程的t.inhertableThreadLocal赋值。这样就能够解决在线程池中每一个新的任务都能够获得父线程中ThreadLocal中的值而不受其他任务的影响，因为在生命周期完成的时候会自动clear所有的数据。
Alibaba的一个库解决了这个问题: https://github.com/alibaba/transmittable-thread-local

## transmittable-thread-local实现原理

JDK的InheritableThreadLocal类可以完成父线程到子线程的值传递。但对于使用线程池等会缓存线程的组件的情况，线程由线程池创建好，并且线程是缓存起来反复使用的；这时父子线程关系的ThreadLocal值传递已经没有意义，应用需要的实际上是把 任务提交给线程池时的ThreadLocal值传递到 任务执行时。

### 如何使用

这个库最简单的方式是这样使用的,通过简单的修饰，使得提交的runable拥有了上一节所述的功能。具体的API文档详见github，这里不再赘述
```java
TransmittableThreadLocal<String> parent = new TransmittableThreadLocal<String>();
parent.set("value-set-in-parent");
 
Runnable task = new Task("1");
// 额外的处理，生成修饰了的对象ttlRunnable
Runnable ttlRunnable = TtlRunnable.get(task); 
executorService.submit(ttlRunnable);
 
// Task中可以读取, 值是"value-set-in-parent"
String value = parent.get();
``` 
### 原理简述
这个方法TtlRunnable.get(task)最终会调用构造方法，返回的是该类本身，也是一个Runable,这样就完成了简单的装饰。最重要的是在run方法这个地方。
```java
public final class TtlRunnable implements Runnable {
 private final AtomicReference<Map<TransmittableThreadLocal<?>, Object>> copiedRef;
 private final Runnable runnable;
 private final boolean releaseTtlValueReferenceAfterRun;
 
 private TtlRunnable(Runnable runnable, boolean releaseTtlValueReferenceAfterRun) {
 //从父类copy值到本类当中
 this.copiedRef = new AtomicReference<Map<TransmittableThreadLocal<?>, Object>>(TransmittableThreadLocal.copy());
 this.runnable = runnable;//提交的runable,被修饰对象
 this.releaseTtlValueReferenceAfterRun = releaseTtlValueReferenceAfterRun;
 }
 /**  * wrap method {@link Runnable#run()}.  */
 @Override
 public void run() {
 Map<TransmittableThreadLocal<?>, Object> copied = copiedRef.get();
 if (copied == null || releaseTtlValueReferenceAfterRun && !copiedRef.compareAndSet(copied, null)) {
 throw new IllegalStateException("TTL value reference is released after run!");
 }
 //装载到当前线程
 Map<TransmittableThreadLocal<?>, Object> backup = TransmittableThreadLocal.backupAndSetToCopied(copied);
 try {
 runnable.run();//执行提交的task
 } finally {
 //clear
 TransmittableThreadLocal.restoreBackup(backup);
 }
 }
}
```
在上面的使用线程池的例子当中，如果换成这种修饰的方式进行操作，B任务得到的肯定是父线程中ThreadLocal的值，解决了在线程池中InheritableThreadLocal不能解决的问题。

更新父线程ThreadLocal值？
如果线程之间出了要能够得到父线程中的值，同时想更新值怎么办呢？在前面我们有提到，当子线程copy父线程的ThreadLocalMap的时候是浅拷贝的,代表子线程Entry里面的value都是指向的同一个引用，我们只要修改这个引用的同时就能够修改父线程当中的值了,比如这样:
```java
@Override
 public void run() {
 System.out.println("========");
 Span span= inheritableThreadLocal.get();
 System.out.println(span);
 span.name="liuliuliu";//修改父引用为liuliuliu
 inheritableThreadLocal.set(new Span("zhangzhangzhang"));
 System.out.println(inheritableThreadLocal.get());
 }
```
这样父线程中的值就会得到更新了。能够满足父线程ThreadLocal值的实时更新，同时子线程也能共享父线程的值。不过场景倒是不是很常见的样子。

## 参考
- https://blog.csdn.net/a837199685/article/details/52712547