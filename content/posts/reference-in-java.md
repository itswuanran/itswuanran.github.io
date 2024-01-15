---
title: Java语言中的各种引用
date: 2018-03-28 11:05:16
tags:
---
## 强引用
```java
  Object strongReference = new Object();
```
强引用是我们一般最常的方式。在只要strongReference没有脱离作用域，或没有被设置成null，GC都不会回收它。

## 软引用
A soft reference is exactly like a weak reference, except that it is less eager to throw away the object to which it refers. An object which is only weakly reachable (the strongest references to it are WeakReferences) will be discarded at the next garbage collection cycle, but an object which is softly reachable will generally stick around for a while.

SoftReferences aren't required to behave any differently than WeakReferences, but in practice softly reachable objects are generally retained as long as memory is in plentiful supply. This makes them an excellent foundation for a cache, such as the image cache described above, since you can let the garbage collector worry about both how reachable the objects are (a strongly reachable object will never be removed from the cache) and how badly it needs the memory they are consuming.
使用被做为软引用的对象时，先要用SoftReference的get方法来看对象是否被回收。如果返回值为null，那对象已经被回收了。会尽可能长的保留引用直到 JVM 内存不足时才会被回收。
SoftReference 在“弱引用”中属于最强的引用。SoftReference 所指向的对象，当没有强引用指向它时，会在内存中停留一段的时间，垃圾回收器会根据 JVM 内存的使用情况（内存的紧缺程度）以及 SoftReference 的 get() 方法的调用情况来决定是否对其进行回收。

被这种引用指向的对象，如果此对象没要再被其他Strong Reference引用的话，可能在任何时候被GC。虽然是可能在任何时候被GC，但是通常是在可用内存数比较低的时候，并且在程序抛出OutOfMemoryError之前才发生对此对象的GC。SoftReference通常被用作实现Cache的对象引用，如果这个对象被GC了，那么他可以在任何时候再重新被创建。另外，根据JDK文档中介绍，实际JVM的实现是鼓励不回收最近创建和最近使用的对象。SoftReference 类的一个典型用途就是用于内存敏感的高速缓存。

## 弱引用
WeakReference 是弱于 SoftReference 的引用类型。弱引用的特性和基本与软引用相似，区别就在于弱引用所指向的对象只要进行系统垃圾回收，不管内存使用情况如何，永远对其进行回收（get() 方法返回 null）。
弱引用使用 get() 方法取得对象的强引用从而访问目标对象。
一旦系统内存回收，无论内存是否紧张，弱引用指向的对象都会被回收。
弱引用也可以避免 Heap 内存不足所导致的异常。
弱引用和软引用没有太大的区别，唯一的区别就是不会像软引用那样，在一定时间没有被访问后，才会被回收，而是只要有GC执行，就会回收。
如果一个被WeakReference引用的对象，当没要任何SoftReference和StrongReference引用时，立即会被GC。和SoftReference的区别是：WeakReference对象是被eagerly collected，即一旦没要任何SoftReference和StrongReference引用，立即被清楚；而只被SoftReference引用的对象，不回立即被清楚，只有当内存不够，即将发生OutOfMemoryError时才被清除，而且是先清除不常用的。SoftReference适合实现Cache用。WeakReference 类的一个典型用途就是规范化映射（ canonicalized mapping ）

## 幽灵引用（虚引用）
A phantom reference is quite different than either SoftReference or WeakReference. Its grip on its object is so tenuous that you can't even retrieve the object -- its get() method always returns null. The only use for such a reference is keeping track of when it gets enqueued into a ReferenceQueue, as at that point you know the object to which it pointed is dead. How is that different from WeakReference, though?

The difference is in exactly when the enqueuing happens.WeakReferences are enqueued as soon as the object to which they point becomes weakly reachable. This is beforefinalization or garbage collection has actually happened; in theory the object could even be "resurrected" by an unorthodox finalize() method, but the WeakReference would remain dead. PhantomReferences are enqueued only when the object is physically removed from memory, and theget() method always returns nullspecifically to prevent you from being able to "resurrect" an almost-dead object.

What good are PhantomReferences? I'm only aware of two serious cases for them: first, they allow you to determine exactly when an object was removed from memory. They are in fact the only way to determine that. This isn't generally that useful, but might come in handy in certain very specific circumstances like manipulating large images: if you know for sure that an image should be garbage collected, you can wait until it actually is before attempting to load the next image, and therefore make the dreaded OutOfMemoryError less likely.

Second, PhantomReferences avoid a fundamental problem with finalization: finalize() methods can "resurrect" objects by creating new strong references to them. So what, you say? Well, the problem is that an object which overridesfinalize() must now be determined to be garbage in at least two separate garbage collection cycles in order to be collected. When the first cycle determines that it is garbage, it becomes eligible for finalization. Because of the (slim, but unfortunately real) possibility that the object was "resurrected" during finalization, the garbage collector has to run again before the object can actually be removed. And because finalization might not have happened in a timely fashion, an arbitrary number of garbage collection cycles might have happened while the object was waiting for finalization. This can mean serious delays in actually cleaning up garbage objects, and is why you can getOutOfMemoryErrors even when most of the heap is garbage.

With PhantomReference, this situation is impossible -- when a PhantomReference is enqueued, there is absolutely no way to get a pointer to the now-dead object (which is good, because it isn't in memory any longer). BecausePhantomReference cannot be used to resurrect an object, the object can be instantly cleaned up during the first garbage collection cycle in which it is found to be phantomly reachable. You can then dispose whatever resources you need to at your convenience.

Arguably, the finalize() method should never have been provided in the first place. PhantomReferencesare definitely safer and more efficient to use, and eliminatingfinalize() would have made parts of the VM considerably simpler. But, they're also more work to implement, so I confess to still using finalize() most of the time. The good news is that at least you have a choice.

```java
ReferenceQueue queue = new ReferenceQueue();
PhantomReference phantomReference= new PhantomReference(new Object(), queue);
```
说明：

虚引用和弱引用非常相似，但不同的一点是，  用PhantomReference的get方法时，返回值永远为null，也就是说你永远无法通过PhantomReference来得到它的引用。而且它必须和ReferenceQueue一起使用，当垃圾收集器确定了某个对象是虚可及对象时， PhantomReference 对象就被放在它的 ReferenceQueue 上。还有一个不同点是，放入RererenceQueue的时机。当弱引用一旦达到可以被回收的状态时，就会被加入到队列里。但这种状态的对象可能会因为对象的finalize()方法里“复活”（或者说回收的更慢）。而PhantomReference对象被放到RererenceQueue的时候，实际上它已经被从物理内存里清除掉了。它的get方法永远返回null，也防止你”复活“它（或者说妨碍它被释放掉）

问题来了，那我们能用它来做什么呢？看了一些文章，有两个地方可用
1，通过ReferenceQueue，跟踪对象什么时候被释放，或者在它被释放后，做一些后续工作。
2，避免因为finalize()方法造成很久才能释放。原因如下：实现了finalize()方法的对象，至少要经过两个GC周期才能被释放。
值得注意的是，对于引用回收方面，虚引用类似强引用不会自动根据内存情况自动对目标对象回收，Client 需要自己对其进行处理以防 Heap 内存不足异常。
不像软引用、弱引用会自动回收内存，虚引用的存在（虽然内存还是会被回收）更倾向于发送通知，当一个对象确定会被回收之后（此时虚引用中的引用对象并不能确定是否已经被回收内存了，而软引用和弱引用一定是被回收内存了的），就会向应用程序发送一个通知（进入队列和出队列），“我要被清理了，你们是否要做些什么事情呢？”。
所以，虚引用用来做对象清理工作，比finalize方法再好不过了，不会导致垃圾回收器额外做工作，配合Reference阻塞方法remove就能更及时做清理工作。
当没有StrongReference，SoftReference和WeakReference引用时，随时可被GC。通常和ReferenceQueue联合使用，管理和清除与被引用对象（没有finalize方法）相关的本地资源。

## 参考
- https://blog.csdn.net/reliveIT/article/details/47664029
- https://www.ibm.com/developerworks/cn/java/j-lo-langref/ 文章中错误很多，
- https://community.oracle.com/blogs/enicholas/2006/05/04/understanding-weak-references
- http://www.infoq.com/cn/articles/jvm-source-code-analysis-finalreference
- com.mysql.jdbc.NonRegisteringDriver的内存泄漏