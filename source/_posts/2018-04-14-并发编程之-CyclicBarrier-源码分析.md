---
layout: post
title: 并发编程之-CyclicBarrier-源码分析
date: 2018-04-14 11:11:11.000000000 +09:00
---
## 前言

在之前的介绍 CountDownLatch 的文章中，CountDown 可以实现多个线程协调，在所有指定线程完成后，主线程才执行任务。

但是，CountDownLatch 有个缺陷，这点 JDK 的文档中也说了：他只能使用一次。在有些场合，似乎有些浪费，需要不停的创建 CountDownLatch 实例，JDK  在 CountDownLatch 的文档中向我们介绍了 CyclicBarrier——循环栅栏。具体使用参见文章 [并发编程之 线程协作工具类](http://thinkinjava.cn/article/37)。

## 源码分析

该类结构如下:
![image.png](https://upload-images.jianshu.io/upload_images/4236553-9d1936c1348d5538.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有一个我们常用的方法 await，还有一个内部类，Generation ，仅有一个参数，有什么作用呢？

在 CyclicBarrier 中，有一个 “代” 的概念，因为 CyclicBarrier 是可以复用的，那么每次所有的线程通过了栅栏，就表示一代过去了，就像我们的新年一样。当所有人跨过了元旦，日历就更新了。

为什么需要这个呢？后面我们看源码的时候在细说，现在说有点不太容易懂。

再看看构造方法，有 2 个构造方法：

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

如果使用 CyclicBarrier 就知道了，CyclicBarrier 支持在所有线程通过栅栏的时候，执行一个线程的任务。

parties 属性就是线程的数量，这个数量用来控制什么时候释放打开栅栏，让所有线
程通过。

好了，CyclicBarrier 的最重要的方法就是 await 方法，当执行了这样一个方法，就像是树立了一个栅栏，将线程挡住了，只有所有的线程都到了这个栅栏上，栅栏才会打开。

看看这个方法的实现。

## await 方法实现

代码加注释如下：

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    // 锁住
    lock.lock();
    try {
        // 当前代
        final Generation g = generation;
        // 如果这代损坏了，抛出异常
        if (g.broken)
            throw new BrokenBarrierException();

        // 如果线程中断了，抛出异常
        if (Thread.interrupted()) {
            // 将损坏状态设置为 true
            // 并通知其他阻塞在此栅栏上的线程
            breakBarrier();
            throw new InterruptedException();
        }
        // 获取下标    
        int index = --count;
        // 如果是 0 ,说明到头了
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                // 执行栅栏任务
                if (command != null)
                    command.run();
                ranAction = true;
                // 更新一代,将 count 重置,将 generation 重置.
                // 唤醒之前等待的线程
                nextGeneration();
                // 结束
                return 0;
            } finally {
                // 如果执行栅栏任务的时候失败了,就将栅栏失效
                if (!ranAction)
                    breakBarrier();
            }
        }

        for (;;) {
            try {
                // 如果没有时间限制,则直接等待,直到被唤醒
                if (!timed)
                    trip.await();
                // 如果有时间限制,则等待指定时间
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                // g == generation >> 当前代
                // ! g.broken >>> 没有损坏
                if (g == generation && ! g.broken) {
                    // 让栅栏失效
                    breakBarrier();
                    throw ie;
                } else {
                    // 上面条件不满足,说明这个线程不是这代的.
                    // 就不会影响当前这代栅栏执行逻辑.所以,就打个标记就好了
                    Thread.currentThread().interrupt();
                }
            }
            // 当有任何一个线程中断了,会调用 breakBarrier 方法.
            // 就会唤醒其他的线程,其他线程醒来后,也要抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
            // g != generation >>> 正常换代了
            // 一切正常,返回当前线程所在栅栏的下标
            // 如果 g == generation，说明还没有换代，那为什么会醒了？
            // 因为一个线程可以使用多个栅栏，当别的栅栏唤醒了这个线程，就会走到这里，所以需要判断是否是当前代。
            // 正是因为这个原因，才需要 generation 来保证正确。
            if (g != generation)
                return index;
            // 如果有时间限制,且时间小于等于0,销毁栅栏,并抛出异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

代码虽然长，但整体逻辑还是很简单的。总结一下该方法吧。

1. 首先，每个 CyclicBarrier 都有一个 Lock，想执行 await 方法，就必须获得这把锁。所以，CyclicBarrier 在并发情况下的性能是不高的。

2. 一些线程中断的判断，注意，CyclicBarrier 中，只有有一个线程中断了，其余的线程也会抛出中断异常。并且，**这个 CyclicBarrier 就不能再次使用了。**

3. 每次线程调用一次 await 方法，表示这个线程到了栅栏这里了，那么就将计数器减一。如果计数器到 0 了，表示这是这一代最后一个线程到达栅栏，就尝试执行我们构造方法中输入的任务。最后，将代更新，计数器重置，并唤醒所有之前等待在栅栏上的线程。

4. 如果不是最后一个线程到达栅栏了，就使用 Condition 的 await 方法阻塞线程。如果等待过程中，线程中断了，就抛出异常。这里，注意一下，如果中断的线程的使用 CyclicBarrier 不是这代的，比如，在最后一次线程执行 signalAll 后，并且更新了这个“代”对象。在这个区间，这个线程被中断了，那么，JDK 认为任务已经完成了，就不必在乎中断了，只需要打个标记。所以，catch 里的 else 判断用于极少情况下出现的判断——任务完成，“代” 更新了，突然出现了中断。这个时候，CyclicBarrier 是不在乎的。因为任务已经完成了。

5. 当有一个线程中断了，也会唤醒其他线程，那么就需要判断 broken 状态。 

6. 如果这个线程被其他的 CyclicBarrier 唤醒了，那么 g 肯定等于 generation，这个事件就不能 return 了，而是继续循环阻塞。反之，如果是当前 CyclicBarrier 唤醒的，就返回线程在 CyclicBarrier 的下标。完成了一次冲过栅栏的过程。

## 总结

从 await 方法看，CyclicBarrier 还是比较简单的，JDK 的思路就是：设置一个计数器，线程每调用一次计数器，就减一，并使用  Condition 阻塞线程。当计数器是0的时候，就唤醒所有线程，并尝试执行构造函数中的任务。由于 CyclicBarrier 是可重复执行的，所以，就需要重置计数器。

CyclicBarrier 还有一个重要的点，就是 generation 的概念，由于每一个线程可以使用多个 CyclicBarrier，每个 CyclicBarrier 又都可以唤醒线程，那么就需要用代来控制，如果代不匹配，就需要重新休眠。同时，这个代还记录了线程的中断状态，如果任何线程中断了，那么所有的线程都会抛出中断异常，并且 CyclicBarrier 不再可用了。

总而言之，CyclicBarrier 是依靠一个计数器实现的，内部有一个 count 变量，每次调用都会减一。当一次完整的栅栏活动结束后，计数器重置，这样，就可以重复利用了。

而他和 CountDownLatch 的区别在于，CountDownLatch 只能使用一次就 over 了，CyclicBarrier 能使用多次，可以说功能类似，CyclicBarrier 更强大一点。并且 CyclicBarrier 携带了一个在栅栏处可以执行的任务。更加灵活。

下面来一张图，说说 CyclicBarrier 的流程。和 CountDownLatch 类似：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-98429ec758d7f705.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)































