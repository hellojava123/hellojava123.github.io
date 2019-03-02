---
layout: post
title: 并发编程——-LinkedTransferQueue
date: 2018-04-22 11:11:11.000000000 +09:00
---
## 1. 前言

Java 中总的算起来有 8 种阻塞队列。

![](https://upload-images.jianshu.io/upload_images/4236553-e05e37fd57cd9fd3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们分析了：
 * [并发编程之 SynchronousQueue 核心源码分析](http://thinkinjava.cn/article/95)
 * [并发编程之 ConcurrentLinkedQueue 源码剖析](http://thinkinjava.cn/article/47)
 * [并发编程之 LinkedBolckingQueue 源码剖析](http://thinkinjava.cn/article/45)
 * 在  [并发编程 —— ScheduledThreadPoolExecutor](http://thinkinjava.cn/article/110) 中顺带分析了 DelayWorkQueue。

`ArrayBlockingQueue` 数组队列，我们在 [使用 ReentrantLock 和 Condition 实现一个阻塞队列](http://thinkinjava.cn/article/97) 看过了 JDK 写的一个例子，就是该类的基本原理和实现。楼主不准备分析了。

`LinkedBlockingDeque `是一个双向链表的队列。常用于 “工作窃取算法”，有机会再分析。

`DelayQueue` 是一个支持延时获取元素的无界阻塞队列。内部用 `PriorityQueue` 实现。有机会再分析。

`PriorityBlockingQueue` 是一个支持优先级的无界阻塞队列，和 `DelayWorkQueue` 类似。有机会再分析。

今天要分析的是剩下的一个比较有意思的队列：`LinkedTransferQueue`。

为什么说有意思呢？他可以算是 `LinkedBolckingQueue` 和 `SynchronousQueue` 和合体。

我们知道 `SynchronousQueue` 内部无法存储元素，当要添加元素的时候，需要阻塞，不够完美，`LinkedBolckingQueue` 则内部使用了大量的锁，性能不高。

两两结合，岂不完美？性能又高，又不阻塞。

我们一起来看看。

## 2. LinkedTransferQueue 介绍

![image.png](https://upload-images.jianshu.io/upload_images/4236553-c6909b571f094dcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类实现了一个 TransferQueue。该接口定义了几个方法：

```java
public interface TransferQueue<E> extends BlockingQueue<E> {
    // 如果可能，立即将元素转移给等待的消费者。 
    // 更确切地说，如果存在消费者已经等待接收它（在 take 或 timed poll（long，TimeUnit）poll）中，则立即传送指定的元素，否则返回 false。
    boolean tryTransfer(E e);

    // 将元素转移给消费者，如果需要的话等待。 
    // 更准确地说，如果存在一个消费者已经等待接收它（在 take 或timed poll（long，TimeUnit）poll）中，则立即传送指定的元素，否则等待直到元素由消费者接收。
    void transfer(E e) throws InterruptedException;

    // 上面方法的基础上设置超时时间
    boolean tryTransfer(E e, long timeout, TimeUnit unit) throws InterruptedException;

    // 如果至少有一位消费者在等待，则返回 true
    boolean hasWaitingConsumer();

    // 返回等待消费者人数的估计值
    int getWaitingConsumerCount();
}
```

相比较普通的阻塞队列，增加了这么几个方法。

## 3. 关键源码分析

阻塞队列不外乎` put ，take，offer ，poll `等方法，再加上` TransferQueue `的 几个 `tryTransfer`  方法。我们看看这几个方法的实现。

`put `方法：

```java
public void put(E e) {
     xfer(e, true, ASYNC, 0);
}
```

`take `方法：

```java
public E take() throws InterruptedException {
    E e = xfer(null, false, SYNC, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
```

`offer` 方法：

```java
public boolean offer(E e) {
    xfer(e, true, ASYNC, 0);
    return true;
}
```

`poll` 方法：

```java
public E poll() {
    return xfer(null, false, NOW, 0);
}
```

`tryTransfer` 方法：

```java
public boolean tryTransfer(E e) {
    return xfer(e, true, NOW, 0) == null;
}
```


`transfer` 方法：

```java
public void transfer(E e) throws InterruptedException {
    if (xfer(e, true, SYNC, 0) != null) {
        Thread.interrupted(); // failure possible only due to interrupt
        throw new InterruptedException();
    }
}
```

可怕，所有方法都指向了` xfer` 方法，只不过传入的不同的参数。

第一个参数，如果是 `put` 类型，就是实际的值，反之就是 null。
第二个参数，是否包含数据，put 类型就是 true，take 就是 false。
第三个参数，执行类型，有立即返回的` NOW`，有异步的` ASYNC`，有阻塞的` SYNC`， 有带超时的 `TIMED`。
第四个参数，只有在 `TIMED `类型才有作用。

So，这个类的关键方法就是 xfer 方法了。

## 4. xfer 方法分析

源码加注释：

```java
private E xfer(E e, boolean haveData, int how, long nanos) {
    if (haveData && (e == null))
        throw new NullPointerException();
    Node s = null;                        // the node to append, if needed

    retry:
    for (;;) {                            // restart on append race
        // 从  head 开始
        for (Node h = head, p = h; p != null;) { // find & match first node
            // head 的类型。
            boolean isData = p.isData;
            // head 的数据
            Object item = p.item;
            // item != null 有 2 种情况,一是 put 操作, 二是 take 的 itme 被修改了(匹配成功)
            // (itme != null) == isData 要么表示 p 是一个 put 操作, 要么表示 p 是一个还没匹配成功的 take 操作
            if (item != p && (item != null) == isData) { 
                // 如果当前操作和 head 操作相同，就没有匹配上，结束循环，进入下面的 if 块。
                if (isData == haveData)   // can't match
                    break;
                // 如果操作不同,匹配成功, 尝试替换 item 成功,
                if (p.casItem(item, e)) { // match
                    // 更新 head
                    for (Node q = p; q != h;) {
                        Node n = q.next;  // update by 2 unless singleton
                        if (head == h && casHead(h, n == null ? q : n)) {
                            h.forgetNext();
                            break;
                        }                 // advance and retry
                        if ((h = head)   == null ||
                            (q = h.next) == null || !q.isMatched())
                            break;        // unless slack < 2
                    }
                    // 唤醒原 head 线程.
                    LockSupport.unpark(p.waiter);
                    return LinkedTransferQueue.<E>cast(item);
                }
            }
            // 找下一个
            Node n = p.next;
            p = (p != n) ? n : (h = head); // Use head if p offlist
        }
        // 如果这个操作不是立刻就返回的类型    
        if (how != NOW) {                 // No matches available
            // 且是第一次进入这里
            if (s == null)
                // 创建一个 node
                s = new Node(e, haveData);
            // 尝试将 node 追加对队列尾部，并返回他的上一个节点。
            Node pred = tryAppend(s, haveData);
            // 如果返回的是 null, 表示不能追加到 tail 节点,因为 tail 节点的模式和当前模式相反.
            if (pred == null)
                // 重来
                continue retry;           // lost race vs opposite mode
            // 如果不是异步操作(即立刻返回结果)
            if (how != ASYNC)
                // 阻塞等待匹配值
                return awaitMatch(s, pred, e, (how == TIMED), nanos);
        }
        return e; // not waiting
    }
}
```


代码有点长，其实逻辑很简单。

逻辑如下: 
找到 `head` 节点,如果 `head` 节点是匹配的操作,就直接赋值,如果不是,添加到队列中。

注意：队列中永远只有一种类型的操作,要么是 `put` 类型, 要么是 `take` 类型.

整个过程如下图：


![image.png](https://upload-images.jianshu.io/upload_images/4236553-d60a5b0368dd8d8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



相比较 `SynchronousQueue` 多了一个可以存储的队列，相比较 `LinkedBlockingQueue` 多了直接传递元素，少了用锁来同步。

性能更高，用处更大。


## 5. 总结

`LinkedTransferQueue `是 `SynchronousQueue` 和 `LinkedBlockingQueue` 的合体，性能比 `LinkedBlockingQueue` 更高（没有锁操作），比 `SynchronousQueue `能存储更多的元素。

当 `put` 时，如果有等待的线程，就直接将元素 “交给” 等待者， 否则直接进入队列。

`put `和 `transfer` 方法的区别是，put 是立即返回的， transfer 是阻塞等待消费者拿到数据才返回。`transfer `方法和 `SynchronousQueue `的 put 方法类似。









