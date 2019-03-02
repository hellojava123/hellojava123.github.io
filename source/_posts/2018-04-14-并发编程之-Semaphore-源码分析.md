---
layout: post
title: 并发编程之-Semaphore-源码分析
date: 2018-04-14 11:11:11.000000000 +09:00
---
## 前言

并发 JUC 包提供了很多工具类，比如之前说的 CountDownLatch，CyclicBarrier ，今天说说这个 Semaphore——信号量，关于他的使用请查看往期文章[并发编程之 线程协作工具类](http://thinkinjava.cn/article/37)，今天的任务就是从源码层面分析一下他的原理。

## 源码分析

如果先不看源码，根据以往我们看过的 CountDownLatch CyclicBarrier 的源码经验来看，Semaphore 会怎么设计呢？

首先，他要实现多个线程线程同时访问一个资源，类似于共享锁，并且，要控制进入资源的线程的数量。

如果根据 JDK 现有的资源，我们是否可以使用 AQS 的 state 变量来控制呢？类似 CountDownLatch 一样，有几个线程我们就为这个 state 变量设置为几，当 state 达到了阈值，其他线程就不能获取锁了，就需要等待。当 Semaphore 调用 release 方法的时候，就释放锁，将 state 减一，并唤醒 AQS 上的线程。

以上，就是我们的猜想，那我们看看 JDK 是不是和我们想的一样。

首先看看 Semaphore 的 UML 结构：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-0a842485b2e18ca3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

内部有 3 个类，继承了 AQS。一个公平锁，一个非公平锁，这点和 ReentrantLock 一摸一样。

看看他的构造器：

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

两个构造器，两个参数，一个是许可线程数量，一个是是否公平锁，默认非公平。

而 Semaphore 有 2 个重要的方法，也是我们经常使用的 2 个方法：

```jav
semaphore.acquire();
// doSomeing.....
semaphore.release();
```

acquire 和 release 方法，我们今天重点看这两个方法的源码，一窥 Semaphore 的全貌。

## acquire 方法源码分析

代码如下：

```java
public void acquire() throws InterruptedException {
	// 尝试获取一个锁
    sync.acquireSharedInterruptibly(1);
}

// 这是抽象类 AQS 的方法
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 如果小于0，就获取锁失败了。加入到AQS 等待队列中。
    // 如果大于0，就直接执行下面的逻辑了。不用进行阻塞等待。
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

```java
// 这是抽象父类 Sync 的方法，默认是非公平的
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
```

```java
// 非公平锁的释放锁的方法
final int nonfairTryAcquireShared(int acquires) {
	// 死循环
    for (;;) {
    	// 获取锁的状态
        int available = getState();
        int remaining = available - acquires;
        // state 变量是否还足够当前获取的
        // 如果小于 0，获取锁就失败了。
        // 如果大于 0，就循环尝试使用 CAS 将 state 变量更新成减去输入参数之后的。
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

这里的释放就是对 state 变量减一（或者更多）的。

返回了剩余的 state 大小。

当返回值小于 0 的时候，说明获取锁失败了，那么就需要进入 AQS 的等待队列了。代码入下：

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 添加一个节点 AQS 队列尾部
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
    	// 死循环
        for (;;) {
        	// 找到新节点的上一个节点
            final Node p = node.predecessor();
            // 如果这个节点是 head，就尝试获取锁
            if (p == head) {
            	// 继续尝试获取锁，这个方法是子类实现的
                int r = tryAcquireShared(arg);
                // 如果大于0，说明拿到锁了。
                if (r >= 0) {
                	// 将 node 设置为 head 节点
                	// 如果大于0，就说明还有机会获取锁，那就唤醒后面的线程，称之为传播
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            // 如果他的上一个节点不是 head，就不能获取锁
            // 对节点进行检查和更新状态，如果线程应该阻塞，返回 true。
            if (shouldParkAfterFailedAcquire(p, node) &&
            	// 阻塞 park，并返回是否中断，中断则抛出异常
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
        	// 取消节点
            cancelAcquire(node);
    }
}
```

总的逻辑就是：
1. 创建一个分享类型的 node 节点包装当前线程追加到 AQS 队列的尾部。

2. 如果这个节点的上一个节点是 head ，就是尝试获取锁，获取锁的方法就是子类重写的方法。如果获取成功了，就将刚刚的那个节点设置成 head。

3. 如果没抢到锁，就阻塞等待。


##  release 方法源码分析

该方法用于释放锁，代码如下：

```java
public void release() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
	// 死循环释放成功
    if (tryReleaseShared(arg)) {
    	// 唤醒 AQS 等待对列中的节点，从 head 开始	
        doReleaseShared();
        return true;
    }
    return false;
}
// Sync extends AbstractQueuedSynchronizer 
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        // 对 state 变量 + 1
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

这里释放锁的逻辑写在了抽象类 Sync 中。逻辑简单，就是对 state 变量做加法。

在加法成功后，执行 `doReleaseShared `方法，这个方法是 AQS 的。

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
            	// 设置 head 的等待状态为 0 ，并唤醒 head 上的线程
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 成功设置成 0 之后，将 head 状态设置成传播状态
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

该方法的主要作用就是从 AQS 的 head 节点开始唤醒线程，注意，这里唤醒是 head 节点的下一个节点，需要和 `doAcquireSharedInterruptibly `方法对应，因为 `doAcquireSharedInterruptibly` 方法唤醒的当前节点的上一个节点，也就是 head 节点。

至此，释放 state 变量，唤醒 AQS 头节点结束。

## 总结

总结一下 Semaphore 的原理吧。

总的来说，Semaphore 就是一个共享锁，通过设置 state 变量来实现对这个变量的共享。当调用 acquire 方法的时候，state 变量就减去一，当调用 release 方法的时候，state 变量就加一。当 state 变量为 0 的时候，别的线程就不能进入代码块了，就会在 AQS 中阻塞等待。



