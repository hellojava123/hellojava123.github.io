---
layout: post
title: 并发编程之-CountDown-源码分析
date: 2018-04-13 11:11:11.000000000 +09:00
---
## 前言

Doug Lea 大神在 JUC 包中为我们准备了大量的多线程工具，其中包括 CountDownLatch ，名为`倒计时门栓`，好像不太好理解。不过，今天的文章之后，我们就彻底理解了。

## 如何使用？

在 JDK 的文档中，带有 2 个例子，我们使用其中一个，测试代码如下：

```java
class Driver2 {

  public static void main(String[] args) throws InterruptedException {
    CountDownLatch doneSignal = new CountDownLatch(10);
    Executor e = Executors.newFixedThreadPool(10);
    for (int i = 0; i < 10; ++i) {
      e.execute(new WorkerRunnable(doneSignal, i));
    }

    doneSignal.await();           // wait for all to finish
    System.err.println("work");
  }
}

class WorkerRunnable implements Runnable {

  private final CountDownLatch doneSignal;
  private final int i;

  WorkerRunnable(CountDownLatch doneSignal, int i) {
    this.doneSignal = doneSignal;
    this.i = i;
  }

  public void run() {
    doWork(i);
    doneSignal.countDown();
  }

  void doWork(int i) {
    System.out.println("work");
  }
}
```

上面的代码中，我们创建了 1 个 CountDowmLatch 对象，在主线程和另外 10 个线程中使用，主线程调用了他的 await 方法，子线程调用了 countDown 方法。

最后输出结果如下：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-3ca8f6c011c48350.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

大部分时候，你会得到上面的结果，这是正常的情况，但也可能你会得到下面的结果：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-531fa3e2e14a127b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这看起来不正常。因为我们需要的结果是：主线程最后打印。什么原因导致的呢？其实是由于时间太快，控制台打印的顺序和实际顺序不同，我们可以在后面加个纳秒参数，就能够看出来了。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-7aa382387277e4cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**从纳秒数就能够看出来，主线程是最后执行的。**

通过一幅图看看这个 demo 的整体执行顺序。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-529b3110d4c15550.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图中，主线程会先执行 await 方法，这个方法会挂起当前线程，相当于 wait 方法。而子线程会陆续执行任务，并执行 countDown 方法，countDown 方法每次执行都会将计数器减 1， 当计数器变成 0 的时候，就会唤醒主线程，主线程开始执行自己的任务。不知道这个图画的是否明显，但楼主尽力了。。。。


好了，知道了如何使用，就来看看源码实现吧。

## 源码实现

首先看看这个类的结构：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-c4d01befdc5b1d34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类是一个独立的类，没有继承别的类，有一个内部类 Sync，这个类继承了 AQS 抽象类，其实，在之前的文章中，我们说过，AQS 是 JUC 所有锁的实现，定义了锁的基本操作。这个内部类重写了 tryAcquireShared 方法和 tryReleaseShared 方法。


然后呢？我们看看构造方法。

##### 构造方法：

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

内部实现还是继承了 AQS 的 Sync 类。

Sync 构造方法：

```java
Sync(int count) {
    setState(count);
}
protected final void setState(int newState) {
    state = newState;
}
/**
 * The synchronization state.
 */
private volatile int state;
```

设置了这个 State 变量，我们之前分析过 AQS 的源码，这个变量可以说是 AQS 实现的核心，通过控制这个变量，能够实现共享共享锁或者独占锁。

那么，如果让我们来设计这个CountDownLatch ，我们该如何设计呢？

事实上，很简单，我们只需要对 state 变量进行减 1 操作，直到这个变量变成 0，我们就唤醒主线程。

不知道 Doug Lea 是不是这么设计的？我们去看看。




##### await 方法

主线程会调用这个方法，让自己阻塞，直到被唤醒。

看看这个方法的实现：

```java
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

```java
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}
```

await 方法调用的是 Sync 的 tryAcquireSharedNanos 方法，方法也贴在上面了。该方法会先调用 tryAcquireShared 方法，如果返回值不是大于等于 0 ，说明当前线程不能获取锁，那么就调用 doAcquireSharedNanos 方法。这个方法内部会将当前线程挂起，直到 state 变成 0，才会被唤醒。

而 tryAcquireShared 方法是需要子类自己实现的。我们看看 CountDown 是如何实现的：

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

很简单，就是获取 state 变量，也就是构造方法中设置的参数。


doAcquireSharedNanos 方法的是如何将当前线程挂起的呢？

代码如下：

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 创建一个 node 对象，对象中有个属性就是当前线程对象。并将这个 node 添加进队列尾部。
    final Node node = addWaiter(Node.SHARED);
    // 中断失败标记
    boolean failed = true;
    try {
        for (;;) {
            // 找到这个 node 的上一个节点
            final Node p = node.predecessor();
            // 如果上一个节点是 head，说明他前面已经没有线程阻挡他获取锁了。
            if (p == head) {
                // 获取锁的状态
                int r = tryAcquireShared(arg);
                // 如果大于等于0，说明可以获取锁
                if (r >= 0) {
                    // 将包装当前线程的 node 设置为 head.
                    setHeadAndPropagate(node, r);
                    // 设置他的 next 是 null，让 GC 回收
                    p.next = null; // help GC
                    // 没有发生错误，不必执行下面的取消操作
                    failed = false;
                    return;
                }
            }
            // 如果他的前面的节点的状态时 -1，那么当前线程就需要等待。
            // 调用 parkAndCheckInterrupt 等待，如果等待过程中被中断了，抛出异常
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)  
            // 如果发生了中断异常，则取消获取锁。
            cancelAcquire(node);
    }
}
```

上面的代码写了很多注释，总的来说，逻辑如下：
1. 将当前线程包装成一个 Node 对象，加入到 AQS 的队列尾部。
2. 如果他前面的 node 是 head ，便可以尝试获取锁了。
3. 如果不是，则阻塞等待，调用的是 LockSupport.park(this); 

**CountDown 的 await 方法就是通过 AQS 的锁机制让主线程阻塞等待。而锁的实现就是通过构造器中设置的 state 变量来控制的。当 state 是 0 的时候，就可以获取锁。然后执行后面的逻辑。**


知道了 await 方法，CountDown 方法应该能猜个大概了。

##### countDown 方法

代码如下：

```java
public void countDown() {
    sync.releaseShared(1);
}
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

调用了 Sync 的 releaseShared 方法，也就是父类 AQS 的方法，AQS 需要子类实现 tryReleaseShared 方法。看看 CountDownLatch 是怎么实现的：

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

该方法很简单，就是将 state 变量减 1，只要减过之后， state 不是 0，就返回 fasle。


回到 releaseShared 方法中，当 tryReleaseShared 返回值是 true 时，也就是 state 是 0，就需要执行 doReleaseShared 方法 ，唤醒阻塞在 CountDown 上的线程了。

唤醒代码如下：

```java
private void doReleaseShared() {

    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

只要队列中 head 节点不是 null，且和 tail 不相等，并且状态是 -1，使用 CAS 将状态修改成 0，如果成功，唤醒当前线程。当前线程就会在 doAcquireSharedInterruptibly 方法中苏醒，再次尝试获取锁，只要他的上一个节点是 head，也就是没有人和他争抢锁，并且 state 是 0，就能够成功获取到锁，继续执行下面的逻辑，不再继续阻塞。


而我们 CountDownLatch 的主线程也就可以被唤醒从而继续执行了。

## 总结

总的来说，CountDownLatch 还是比较简单的。说白了就是通过共享锁实现的。在我们的代码中，只有一个线程会阻塞，那就是我们的主线程， 其余的线程就是在不停的释放 state 变量，直到为 0。从 AQS 的角度来讲，整个工作流程如下图：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-d155b56e1cfb5436.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


简单的一个流程图，CountDownLatch 就是通过使用 AQS 的机制来实现`倒计时门栓`的。


good luck！！！！

