---
title: 高并发下的QPS计算
date: 2018-06-27 11:57:30
tags:
- QPS
categories:
- sentinel
---

## 术语说明
QPS = req/sec = 请求数/秒
## 【QPS计算PV和机器的方式】
QPS统计方式 [一般使用 http_load 进行统计]QPS = 总请求数 / ( 进程总数 *   请求时间 )
QPS: 单个进程每秒请求服务器的成功次数单台服务器每天PV
#### 计算
公式1：每天总PV = QPS * 3600 * 6
公式2：每天总PV = QPS * 3600 * 8
### 服务器计算
服务器数量 =   ceil( 每天总PV / 单台服务器每天总PV )
###【峰值QPS和机器计算公式】
原理：每天80%的访问集中在20%的时间里，这20%时间叫做峰值时间
公式：( 总PV数 * 80% ) / ( 每天秒数 * 20% ) = 峰值时间每秒请求数(QPS)
机器：峰值时间每秒QPS / 单台机器的QPS   = 需要的机器
问：每天300w PV 的在单台机器上，这台机器需要多少QPS？
答：( 3000000 * 0.8 ) / (86400 * 0.2 ) = 139 (QPS)
问：如果一台机器的QPS是58，需要几台机器来支持？
答：139 / 58 = 3


## 分片
把一段时间分成若干片，如把1s分成10片，那么每片统计当前100ms内的数据，然后当前qps则为当前分片往前推10格，再求和，即为当前的qps。
那么问题来了，在分片的交接时刻，需要为新的分片创建对应的对象，若不加控制的话，直接加锁，会导致所有的线程等待（只有一个线程去创建当前bucket）。但sentinel的模式则是若发现要创建新的bucket，则让一个线程去创建，别的线程则取出上一个bucket进行处理（牺牲了一点时刻准确度，但换来了性能的大量提升）

Hystrix内部维护一个队列来存放bucket对象： 

```java 
    /* package for testing */Bucket getCurrentBucket() {
        long currentTime = time.getCurrentTimeInMillis();

        /* a shortcut to try and get the most common result of immediately finding the current bucket */

        /**
         * Retrieve the latest bucket if the given time is BEFORE the end of the bucket window, otherwise it returns NULL.
         * 
         * NOTE: This is thread-safe because it's accessing 'buckets' which is a LinkedBlockingDeque
         */
        Bucket currentBucket = buckets.peekLast();
        if (currentBucket != null && currentTime < currentBucket.windowStart + this.bucketSizeInMillseconds) {
            // if we're within the bucket 'window of time' return the current one
            // NOTE: We do not worry if we are BEFORE the window in a weird case of where thread scheduling causes that to occur,
            // we'll just use the latest as long as we're not AFTER the window
            return currentBucket;
        }

        /* if we didn't find the current bucket above, then we have to create one */

        /**
         * The following needs to be synchronized/locked even with a synchronized/thread-safe data structure such as LinkedBlockingDeque because
         * the logic involves multiple steps to check existence, create an object then insert the object. The 'check' or 'insertion' themselves
         * are thread-safe by themselves but not the aggregate algorithm, thus we put this entire block of logic inside synchronized.
         * 
         * I am using a tryLock if/then (http://download.oracle.com/javase/6/docs/api/java/util/concurrent/locks/Lock.html#tryLock())
         * so that a single thread will get the lock and as soon as one thread gets the lock all others will go the 'else' block
         * and just return the currentBucket until the newBucket is created. This should allow the throughput to be far higher
         * and only slow down 1 thread instead of blocking all of them in each cycle of creating a new bucket based on some testing
         * (and it makes sense that it should as well).
         * 
         * This means the timing won't be exact to the millisecond as to what data ends up in a bucket, but that's acceptable.
         * It's not critical to have exact precision to the millisecond, as long as it's rolling, if we can instead reduce the impact synchronization.
         * 
         * More importantly though it means that the 'if' block within the lock needs to be careful about what it changes that can still
         * be accessed concurrently in the 'else' block since we're not completely synchronizing access.
         * 
         * For example, we can't have a multi-step process to add a bucket, remove a bucket, then update the sum since the 'else' block of code
         * can retrieve the sum while this is all happening. The trade-off is that we don't maintain the rolling sum and let readers just iterate
         * bucket to calculate the sum themselves. This is an example of favoring write-performance instead of read-performance and how the tryLock
         * versus a synchronized block needs to be accommodated.
         */
        if (newBucketLock.tryLock()) {
            try {
                if (buckets.peekLast() == null) {
                    // the list is empty so create the first bucket
                    Bucket newBucket = new Bucket(currentTime);
                    buckets.addLast(newBucket);
                    return newBucket;
                } else {
                    // We go into a loop so that it will create as many buckets as needed to catch up to the current time
                    // as we want the buckets complete even if we don't have transactions during a period of time.
                    for (int i = 0; i < numberOfBuckets; i++) {
                        // we have at least 1 bucket so retrieve it
                        Bucket lastBucket = buckets.peekLast();
                        if (currentTime < lastBucket.windowStart + this.bucketSizeInMillseconds) {
                            // if we're within the bucket 'window of time' return the current one
                            // NOTE: We do not worry if we are BEFORE the window in a weird case of where thread scheduling causes that to occur,
                            // we'll just use the latest as long as we're not AFTER the window
                            return lastBucket;
                        } else if (currentTime - (lastBucket.windowStart + this.bucketSizeInMillseconds) > timeInMilliseconds) {
                            // the time passed is greater than the entire rolling counter so we want to clear it all and start from scratch
                            reset();
                            // recursively call getCurrentBucket which will create a new bucket and return it
                            return getCurrentBucket();
                        } else { // we're past the window so we need to create a new bucket
                            // create a new bucket and add it as the new 'last'
                            buckets.addLast(new Bucket(lastBucket.windowStart + this.bucketSizeInMillseconds));
                            // add the lastBucket values to the cumulativeSum
                            cumulativeSum.addBucket(lastBucket);
                        }
                    }
                    // we have finished the for-loop and created all of the buckets, so return the lastBucket now
                    return buckets.peekLast();
                }
            } finally {
                newBucketLock.unlock();
            }
        } else {
            currentBucket = buckets.peekLast();
            if (currentBucket != null) {
                // we didn't get the lock so just return the latest bucket while another thread creates the next one
                return currentBucket;
            } else {
                // the rare scenario where multiple threads raced to create the very first bucket
                // wait slightly and then use recursion while the other thread finishes creating a bucket
                try {
                    Thread.sleep(5);
                } catch (Exception e) {
                    // ignore
                }
                return getCurrentBucket();
            }
        }
    }


class ListState {
    /*
     * this is an AtomicReferenceArray and not a normal Array because we're copying the reference
     * between ListState objects and multiple threads could maintain references across these
     * compound operations so I want the visibility/concurrency guarantees
     */
    private final AtomicReferenceArray<Bucket> data;
    private final int size;//bucket size
    private final int tail;//尾指针
    private final int head;//头指针
    //...
}
```
默认情况下，10s的滚动窗口时间，分10个桶，通过时间移动来创建新的bucket淘汰旧的bucket。
Hystrix维护的队列与一般情况下使用的Queue不同，它必须按时间来添加bucket、变更指针指向（head、tail），同一秒内，通过getCurrentBucket()获取到的bucket应该是同一个，每当时间滚动到下一秒需要一个新的bucket时，通过tryLock来保证只有一个线程去new bucket；整个统计过程中唯一加锁的地方只有新建bucket的时候，并且它的加锁频率是一秒一次（当固定滚动窗口时间不变，分桶数越小加锁频率越低），并不是桶分得越少越好，粒度大小需要依业务的具体情况来权衡。

## LongAdd
## Striped64
数据 striping 就是把逻辑上连续的数据分为多个段，使这一序列的段存储在不同的物理设备上。通过把段分散到多个设备上可以增加访问并发性，从而提升总体的吞吐量。
类似concurrentHashMap的分段锁。


## Sentinel 实现
```java
// ArrayMetric 实现了Metric 接口，同时包含了 MetricsLeapArray数据结构，接口的实现就是通过这个MetricsLeapArray来实现的
// MetricsLeapArray 是从 LeapArray 继承的，所以这一篇的重点就是LeapArray了
public class ArrayMetric implements Metric {
    private final MetricsLeapArray data;

    /**
     * Constructor
     *
     * @param windowLengthInMs a single window bucket's time length in milliseconds.
     * @param intervalInSec    the total time span of this {@link ArrayMetric} in seconds.
     */
    public ArrayMetric(int windowLengthInMs, int intervalInSec) {
        this.data = new MetricsLeapArray(windowLengthInMs, intervalInSec);
    }
}
```

## 新增当前统计数据
```java
    @Override
    public void addSuccess() {
        WindowWrap<MetricBucket> wrap = data.currentWindow();
        wrap.value().addSuccess();
    }
```
## 获取时间窗口内统计数据
```java
    @Override
    public long success() {
        data.currentWindow();
        long success = 0;

        List<MetricBucket> list = data.values();
        for (MetricBucket window : list) {
            success += window.success();
        }
        return success;
    }
```

```java
protected final AtomicReferenceArray<WindowWrap<T>> array;

public LeapArray(int windowLengthInMs, int intervalInSec) {
        this.windowLengthInMs = windowLengthInMs;
        this.intervalInMs = intervalInSec * 1000;
        this.sampleCount = intervalInMs / windowLengthInMs;
        // 初始化容量大小
        this.array = new AtomicReferenceArray<WindowWrap<T>>(sampleCount);
    }


  /**
     * Get window at provided timestamp.
     *
     * @param time a valid timestamp
     * @return the window at provided timestamp
     */
    public WindowWrap<T> currentWindow(long time) {
        long timeId = time / windowLengthInMs;
        // Calculate current index.
        int idx = (int)(timeId % array.length());

        // Cut the time to current window start.
        time = time - time % windowLengthInMs;

        while (true) {
            WindowWrap<T> old = array.get(idx);
            if (old == null) {
                WindowWrap<T> window = new WindowWrap<T>(windowLengthInMs, time, newEmptyBucket());
                if (array.compareAndSet(idx, null, window)) {
                    return window;
                } else {
                    Thread.yield();
                }
            } else if (time == old.windowStart()) {
                return old;
            } else if (time > old.windowStart()) {
                if (updateLock.tryLock()) {
                    try {
                        // if (old is deprecated) then [LOCK] resetTo currentTime.
                        return resetWindowTo(old, time);
                    } finally {
                        updateLock.unlock();
                    }
                } else {
                    Thread.yield();
                }

            } else if (time < old.windowStart()) {
                // Cannot go through here.
                return new WindowWrap<T>(windowLengthInMs, time, newEmptyBucket());
            }
        }
    }
```

## MetricBucket.java
```java
/**
 * Represents metrics data in a period of time window.
 *
 * @author jialiang.linjl
 * @author Eric Zhao
 */
public class MetricBucket {

    private final LongAdder pass = new LongAdder();
    private final LongAdder block = new LongAdder();
    private final LongAdder exception = new LongAdder();
    private final LongAdder rt = new LongAdder();
    private final LongAdder success = new LongAdder();

    private volatile long minRt;

    public MetricBucket() {
        initMinRt();
    }

    private void initMinRt() {
        this.minRt = Constants.TIME_DROP_VALVE;
    }

    /**
     * Clean the adders and reset window to provided start time.
     *
     * @return new clean window
     */
    public MetricBucket reset() {
        pass.reset();
        block.reset();
        exception.reset();
        rt.reset();
        success.reset();
        initMinRt();
        return this;
    }

    public long pass() {
        return pass.sum();
    }

    public long block() {
        return block.sum();
    }

    public long exception() {
        return exception.sum();
    }

    public long rt() {
        return rt.sum();
    }

    public long minRt() {
        return minRt;
    }

    public long success() {
        return success.sum();
    }

    public void addPass() {
        pass.add(1L);
    }

    public void addException() {
        exception.add(1L);
    }

    public void addBlock() {
        block.add(1L);
    }

    public void addSuccess() {
        success.add(1L);
    }

    public void addRT(long rt) {
        this.rt.add(rt);

        // Not thread-safe, but it's okay.
        if (rt < minRt) {
            minRt = rt;
        }
    }
}
```
## 参考
- https://blog.csdn.net/chaofanwei2/article/details/52049709
- https://www.zhihu.com/question/21556347/answer/83666444
- https://my.oschina.net/tigerlene/blog/2250158 源码分析
- https://www.geeksforgeeks.org/window-sliding-technique/ 滑动窗口算法
