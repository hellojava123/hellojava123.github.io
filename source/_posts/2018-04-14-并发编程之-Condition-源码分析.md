---
layout: post
title: 并发编程之-Condition-源码分析
date: 2018-04-14 11:11:11.000000000 +09:00
---
## 前言

Condition 是  Lock 的伴侣，至于如何使用，我们之前也写了一些文章来说，例如  [使用 ReentrantLock 和 Condition 实现一个阻塞队列](http://thinkinjava.cn/article/97)，[并发编程之 Java 三把锁](http://thinkinjava.cn/article/36)， 在这两篇文章中，我们都详细介绍了他们的使用。今天我们就来深入看看源码实现。

## 构造方法

`Condition` 接口有 2 个实现类，一个是 `AbstractQueuedSynchronizer.ConditionObject`，还有一个是 `AbstractQueuedLongSynchronizer.ConditionObject`，都是 AQS 的内部类，该类结构如下：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-0ae2dda402d75fe3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


几个公开的方法：
1. await()
2. await(long time, TimeUnit unit)
3. awaitNanos(long nanosTimeout)
4. awaitUninterruptibly()
5. awaitUntil(Date deadline)
6. signal() 
7. signalAll() 

今天我们重点关注 2 个方法，也是最常用的 2 个方法： await 和 signal。

## await 方法

先贴一波代码加注释：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 创建一个新的节点,追加到 Condition 队列中最后一个节点.
    Node node = addConditionWaiter();
    // 释放这个锁,并唤醒 AQS 队列中一个线程.
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 判断这个节点是否在 AQS 队列上,第一次判断总是返回 false
    while (!isOnSyncQueue(node)) {
        // 第一次总是 park 自己,开始阻塞等待
        LockSupport.park(this);
        // 线程判断自己在等待过程中是否被中断了,如果没有中断,则再次循环,会在 isOnSyncQueue 中判断自己是否在队列上.
        //  isOnSyncQueue 判断当前 node 状态,如果是 CONDITION 状态,或者不在队列上了(JDK 注释说,由于 CAS 操作队列上的节点可能会失败),就继续阻塞.
        //  isOnSyncQueue 判断当前 node 还在队列上且不是 CONDITION 状态了,就结束循环和阻塞.
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            // 如果被中断了,就跳出循环
            break;
    }
    // 当这个线程醒来,会尝试拿锁, 当 acquireQueued 返回 false 就是拿到锁了.
    // interruptMode != THROW_IE >>> 表示这个线程没有成功将 node 入队,但 signal 执行了 enq 方法让其入队了.
    // 将这个变量设置成 REINTERRUPT.
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 如果 node 的下一个等待者不是 null, 则进行清理,清理 Condition 队列上的节点. 
    // 如果是 null ,就没有什么好清理的了.
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 如果线程被中断了,需要抛出异常.或者什么都不做
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

**总结一下这个方法的逻辑:**
* 在 Condition 中, 维护着一个队列,每当执行 await 方法,都会根据当前线程创建一个节点,并添加到尾部.
* 然后释放锁,并唤醒阻塞在锁的 AQS 队列中的一个线程.
* 然后,将自己阻塞.
* 在被别的线程唤醒后, 将刚刚这个节点放到 AQS 队列中.接下来就是那个节点的事情了,比如抢锁.
* 紧接着就会尝试抢锁.接下来的逻辑就和普通的锁一样了,抢不到就阻塞,抢到了就继续执行.

看看详细的源码实现，Condition 是如何添加节点的？addConditionWaiter 方法如下：

```java
// 该方法就是创建一个当前线程的节点,追加到最后一个节点中.
private Node addConditionWaiter() {
    // 找到最后一个节点,放在局部变量中,速度更快
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    // 如果最后一个节点失效了,就清除链表中所有失效节点,并重新赋值 t
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 创建一个当前线程的 node 节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 如果最后一个节点是 null
    if (t == null)
        // 将当前节点设置成第一个节点
        firstWaiter = node;
    else
        // 如果不是 null, 将当前节点追加到最后一个节点
        t.nextWaiter = node;
    // 将当前节点设置成最后一个节点
    lastWaiter = node;
    // 返回
    return node;
}
```
创建一个当前线程的节点,追加到最后一个节点中. 当然，其中还有一个 unlinkCancelledWaiters 方法的调用，当最后一个节点失效了，就需要清理 Condition 队列中无效的节点，代码如下：

```java
// 清除链表中所有失效的节点.
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    // 当 next 正常的时候,需要保存这个 next, 方便下次循环是链接到下一个节点上.
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        // 如果这个节点被取消了
        if (t.waitStatus != Node.CONDITION) {
            // 先将他的 next 节点设置为 null
            t.nextWaiter = null;
            // 如果这是第一次判断 trail 变量
            if (trail == null)
                // 将 next 变量设置为 first, 也就是去除之前的 first(由于是第一次,肯定去除的是 first)
                firstWaiter = next;
            else
                // 如果不是 null,说明上个节点正常,将上个节点的 next 设置为无效节点的 next, 让 t 失效
                trail.nextWaiter = next;
            // 如果 next 是 null, 说明没有节点了,那么就可以将 trail 设置成最后一个节点
            if (next == null)
                lastWaiter = trail;
        }
        // 如果该节点正常,那么就保存这个节点,在下次链接下个节点时使用
        else
            trail = t;
        // 换下一个节点继续循环
        t = next;
    }
}
```

那么又是如何释放锁，并唤醒 AQS 队列中的一个节点上的线程的呢？fullyRelease 方法如下：

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取 state 变量
        int savedState = getState();
        // 如果释放成功,则返回 state 的大小,也就是之前持有锁的线程的数量
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            // 如果释放失败,抛出异常
            throw new IllegalMonitorStateException();
        }
    } finally {
        //释放失败
        if (failed)
            // 将这个节点是指成取消状态.随后将从队列中移除.
            node.waitStatus = Node.CANCELLED;
    }
}
```

而 release 方法中，又是如何操作的呢？代码如下：

```java
//  主要功能,就是释放锁,并唤醒阻塞在锁上的线程.
public final boolean release(int arg) {
	// 如果释放锁成功,返回 true, 可能会抛出监视器异常,即当前线程不是持有锁的线程.
	// 也可能是释放失败,但 fullyRelease 基本能够释放成功.
    if (tryRelease(arg)) {
    	// 释放成功后, 唤醒 head 的下一个节点上的线程.
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    // 释放失败
    return false;
}
```

release 方法主要调用了 tryRelease 方法，该方法就是释放锁的。tryRelease 代码如下：

```java
// 主要功能就是对 state 变量做减法, 如果 state 变成0,则将持有锁的线程设置成 null.
protected final boolean tryRelease(int releases) {
	// 计算 state
    int c = getState() - releases;
    // 如果当前线程不是持有该锁的线程,则抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 如果结果是 0,说明成功释放了锁.
    if (c == 0) {
        free = true;
        // 将持有当前锁的线程设置成 null.
        setExclusiveOwnerThread(null);
    }
    // 设置变量
    setState(c);
    return free;
}
```

好了，我们大概知道了 Condition 是如何释放锁的了，那么又是如何将自己阻塞的呢？在将自己阻塞之前，需要调用 isOnSyncQueue 方法判断，代码如下：

```java
final boolean isOnSyncQueue(Node node) {
    // 如果他的状态不是等地啊,且他的上一个节点是 null, 便不在队列中了
    // 这里判断 == CONDITION,实际上是第一次判断,而后面的判断则是线程醒来后的判断.
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 如果他的 next 不是 null, 说明他还在队列上.
    if (node.next != null) // If has successor, it must be on queue
        return true;
    // 如果从 tail 开始找上一个节点,找到了给定的节点,说明也在队列上.返回 true.
    return findNodeFromTail(node);
}
```

实际上，第一次总是会返回 fasle，从而进入 while 块调用 park 方法，阻塞自己，至此，Condition 成功的释放了所在的 Lock 锁，并将自己阻塞。

虽然阻塞了，但总有人会调用 signal 方法唤醒他，唤醒之后走下面的 if 逻辑，也就是 checkInterruptWhileWaiting 方法，看名字是当等待的时候检查中断状态，代码如下：

```java
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        // transferAfterCancelledWait >>>> 如果将 node 放入 AQS 队列失败,就返回 REINTERRUPT, 成功则返回 THROW_IE
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) : 0;
}
```

当线程中断了，就需要根据调用 transferAfterCancelledWait 方法的返回值来返回不同的常量，该方法内部逻辑是怎么样的呢？

```java
final boolean transferAfterCancelledWait(Node node) {
    // 将 node 的状态设置成 0 成功后,将这个 node 放进 AQS 队列中.
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    // 如果 CAS 失败, 返回 false ,
    // 当 node 不在 AQS 节点上, 就自旋. 直到 enq 方法完成.
    // JDK 认为, 在这个过程中, node 不在 AQS 队列上是少见的,也是暂时的.所以自旋.
    // 如果不自旋的话,后面的逻辑是无法继续执行的. 实际上,自旋是在等待在 signal 中执行 enq 方法让 node 入队.
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

其实是尝试将自己放入到队列中。如果无法放入，就自旋等待 signal 方法放入。

回到 await 方法，继续向下走，执行 3 个 if 块，第一个 if 块，尝试拿锁，为什么？因为这个时候，这个线程已经被唤醒了，而且他在 AQS 的队列中，那么，他就需要在醒的时候，去拿锁。acquireQueued 方法是拿锁的逻辑，代码如下：

```java

// 返回结果:是否被中断了, 当返回 false 就是拿到锁了,反之没有拿到.
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 返回他的上一个节点
            final Node p = node.predecessor();
            // 如果这个节点的上个节点是 head, 且成功获取了锁.
            if (p == head && tryAcquire(arg)) {
                // 将当前节点设置成 head
                setHead(node);
                // 他的上一个节点(head)设置成 null.
                p.next = null; // help GC
                failed = false;
                // 返回 false,没有中断
                return interrupted;
            }
            // shouldParkAfterFailedAcquire >>> 如果没有获取到锁,就尝试阻塞自己等待(上个节点的状态是  -1 SIGNAL).
            // parkAndCheckInterrupt >>>> 返回自己是否被中断了.
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

注意，如果这里拿不到锁，就会在 parkAndCheckInterrupt 方法中阻塞。这里和正常的 AQS 中的队列节点是一摸一样的，没有特殊。

这里有个 tryAcquire 方法需要注意一下： 这个就是尝试拿锁的逻辑所在。代码如下：

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // 如果锁的状态是空闲的.
    if (c == 0) {
        // !hasQueuedPredecessors() >>>  是否含有比自己的等待的时间长的线程, false >> 没有
        // compareAndSetState >>> CAS 设置 state 变量成功
        // 设置当前线程为锁的持有线程成功
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            // 上面 3 个条件都满足, 抢锁成功.
            return true;
        }
    }
    // 如果 state 状态不是0, 且当前线程和锁的持有线程相同,则认为是重入.
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

主要逻辑是设置 state 变量，将锁的持有线程变成自己。这些是在没有比自己等待时间长的线程的情况下发生的。意思是，优先哪些等待时间久的线程拿锁。当然，这里还有一些重入的逻辑。


后面的两个 if 块就简单了，如果 Condition 中还有节点，那么就尝试清理无效的节点，调用的是 unlinkCancelledWaiters 方法，这个方法，我们在上面分析过了，就不再重复分析了。

最后，判断是否中断，执行 reportInterruptAfterWait 方法，这个方法可能会抛出异常，也可能会对当前线程打一个中断标记。

代码如下：

```java
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}

```

好，关于 await 方法就分析完了，可以看到，楼主贴了很多的注释，事实上，都是为了以后能更好的复习，也方便感兴趣的同学在看源码的时候结合我的注释一起分析。

这个方法的总结都在开始说了，就不再重复总结了。后面，我们会结合 signal 方法一起总结。

再来看看 signal 方法实现。



## signal 方法

代码如下：

```java
public final void signal() {
	// 如果当前线程不是持有该锁的线程.抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 拿到 Condition 队列上第一个节点
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

很明显，唤醒策略是从头部开始的。

看看 doSignal(first) 方法实现：


```java
private void doSignal(Node first) {
    do {
    	// 如果第一个节点的下一个节点是 null, 那么, 最后一个节点也是 null.
        if ((firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        // 将 next 节点设置成 null.
        first.nextWaiter = null;
        // 如果修改这个 node 状态为0失败了(也就是唤醒失败), 并且 firstWaiter 不是 null, 就重新循环.
        // 通过从 First 向后找节点,直到唤醒或者没有节点为止.
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

重点在于 transferForSignal 方法，该方法肯定做了唤醒操作。

```java
final boolean transferForSignal(Node node) {
    // 如果不能改变状态,就取消这个 node. 
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    // 将这个 node 放进 AQS 的队列,然后返回他的上一个节点.
    Node p = enq(node);
    int ws = p.waitStatus;
    // 如果上一个节点的状态被取消了, 或者尝试设置上一个节点的状态为 SIGNAL 失败了(SIGNAL 表示: 他的 next 节点需要停止阻塞), 
    // 唤醒输入节点上的线程.
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    // 如果成功修改了 node 的状态成0,就返回 true.
    return true;
}
```

果然，看到了 unpark 操作，该方法先是 CAS 修改了节点状态，如果成功，就将这个节点放到 AQS 队列中，然后唤醒这个节点上的线程。此时，那个节点就会在 await 方法中苏醒，并在执行 checkInterruptWhileWaiting 方法后开始尝试获取锁。


## 总结

总结一下 Condition 执行 await 和 signal 的过程吧。

1. 首先，线程如果想执行 await 方法，必须拿到锁，在 AQS 里面，抢到锁的一般都是 head，然后 head 失效，从队列中删除。

2. 在当前线程（也就是 AQS 的 head）拿到锁后，调用了 await 方法，第一步创建一个 Node 节点，放到 Condition 自己的队列尾部，并唤醒 AQS 队列中的某个（head）节点，然后阻塞自己，等待被 signal 唤醒。

3. 当有其他线程调用了 signal 方法，就会唤醒 Condition 队列中的 first 节点，然后将这个节点放进 AQS 队列的尾部。

4. 阻塞在 await 方法的线程苏醒后，他已经从 Condition 队列总转移到 AQS 队列中了，这个时候，他就是一个正常的 AQS 节点，就会尝试抢锁。并清除 Condition 队列中无效的节点。

下面这张图说明了这几个步骤。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-6f61ca06236ce9fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



好啦，关于 Condition 就说到这里啦，总的来说，就是在 Condition 中新增一个休眠队列来实现的。只要调用 await 方法，就会休眠，进入 Condition 队列，调用 signal 方法，就会从 Condition 队列中取出一个线程并插入到 AQS 队列中，然后唤醒，让这个线程自己去抢锁。




