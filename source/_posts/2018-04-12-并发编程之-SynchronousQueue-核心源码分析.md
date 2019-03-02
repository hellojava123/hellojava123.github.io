---
layout: post
title: 并发编程之-SynchronousQueue-核心源码分析
date: 2018-04-12 11:11:11.000000000 +09:00
---

## 前言

`SynchronousQueue` 是一个普通用户不怎么常用的队列，通常在创建无界线程池（`Executors.newCachedThreadPool()`）的时候使用，也就是那个非常危险的线程池  `^_^`。

它是一个非常特殊的阻塞队列，他的模式是：在 `offer `的时候，如果没有另一个线程在 take 或者 `poll` 的话，就会失败，反之，如果在 `take `或者 `poll `的时候，没有线程在` offer` ，则也会失败，而这种特性，则非常适合用来做高响应并且线程不固定的线程池的` Queue`。所以，在很多高性能服务器中，如果并发很高，这时候，普通的 `LinkedQueue `就会成为瓶颈，性能就会出现毛刺，当换上 `SynchronousQueue `后，性能就会好很多。

今天就看看这个特殊的 Queue 是怎么实现的。友情提示：代码有些小复杂。。。请做好心理准备。

## 源码实现

SynchronousQueue 内部分为公平（队列）和非公平（栈），队列的性能相对而言会好点。构造方法中，就看出来了。默认是非公平的，通常非公平（栈 FIFO）的性能会高那么一点点。

**构造方法**：

```java
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

**offer 方法**

该方法我们通常建议使用带有超时机制的 `offer `方法。

```java 
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}

```

从上面的代码中，可以看到核心方法就是 `transfer `方法。如果该方法返回 `true`，表示，插入成功，如果失败，就返回 `false`。

**poll 方法**

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E e = transferer.transfer(null, true, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}
```

同样的该方法也是调用了 `transfer` 方法。结果返回得到的值或者` null`。区别在于，`offer `方法的 `e` 参数是实体的。而 `poll` 方法 `e` 参数是 `null`，我们猜测，方法内部肯定根据这个做了判断。所以，重点在于` transfer `方法的实现。

而 transferer 有 2 种，队列和栈，我们就研究一种，知晓其原理，另一种有时间在看。

## TransferQueue 源码实现

构造方法：

```java
TransferQueue() {
    QNode h = new QNode(null, false); // initialize to dummy node.
    head = h;
    tail = h;
}
```

构造一个 Node 节点，注释说这是一个加的 node。并赋值给 head 和 tail 节点。形成一个初始化的链表。

看看这个 node：

```java
/** Node class for TransferQueue. */
static final class QNode {
    volatile QNode next;          // next node in queue
    volatile Object item;         // CAS'ed to or from null
    volatile Thread waiter;       // to control park/unpark
    final boolean isData;
}
```

node 持有队列中下一个 node，node 对应的值 value，持有该 node 的线程，拥有 park 或者 unpark，这里用的是 JUC 的工具类 LockSupport，还有一个布尔类型，isData，这个非常重要，需要好好理解，到后面我们会好好讲解。

我们更关注的是这个类的 transfer 方法，该方法是 SynchronousQueue 的核心。

该方法接口定义如下：

```java
/**
 * Performs a put or take. put 或者 take
 *
 * @param e if non-null, the item to be handed to a consumer;
 *          if null, requests that transfer return an item
 *          offered by producer. 
 * @param timed if this operation should timeout
 * @param nanos the timeout, in nanoseconds
 * @return if non-null, the item provided or received; if null,
 *         the operation failed due to timeout or interrupt --
 *         the caller can distinguish which of these occurred
 *         by checking Thread.interrupted.
 */
abstract E transfer(E e, boolean timed, long nanos);
```

注释说道 e 参数的作用：

> 如果 e 不是 null(说明是生产者调用) ，将 item 交给消费者，并返回 e；反之，如果是 null（说明是消费者调用），将生产者提供的 item 返回给消费者。

看看 TransferQueue 类的 transfer 方法实现，楼主写了很多的注释尝试解读：

```java
QNode s = null; // constructed/reused as needed
boolean isData = (e != null);// 当输入的是数据时，isData 就是 ture，表明这个操作是一个输入数据的操作；同理，当调用者输入的是 null，则是在消费数据。

for (;;) {
    QNode t = tail;
    QNode h = head;
    if (t == null || h == null)         // 如果并发导致未"来得及"初始化
        continue;                       // 自旋重来

    // 以下分成两个部分进行

    // 1. 如果当前操作和 tail 节点的操作是一样的；或者头尾相同（表明队列中啥都没有）。
    if (h == t || t.isData == isData) { 
        QNode tn = t.next;
        if (t != tail)                  // 如果 t 和 tail 不一样，说明，tail 被其他的线程改了，重来
            continue;
        if (tn != null) {               // 如果 tail 的 next 不是空。就需要将 next 追加到 tail 后面了。
            advanceTail(t, tn); // 使用 CAS 将 tail.next 变成 tail,        
            continue;
        }
        if (timed && nanos <= 0)        // 时间到了，不等待，返回 null，插入失败，获取也是失败的。
            return null;
        if (s == null) // 如果能走到这里，说明 tail 的 next 是 null，这里的判断是避免重复创建 Qnode 对象。
            s = new QNode(e, isData);// 创建一个新的节点。
        if (!t.casNext(null, s))        // 尝试 CAS 将这个刚刚创建的节点追加到 tail 的 next 节点上.
            continue;// 如果失败，则重来

        advanceTail(t, s); // 当新的节点成功追加到 tail 节点的 next 上了， 就尝试将 tail.next 节点覆盖 tail 节点,称之为推进。
        // s == 新节点，“可能”是新的 tail；e 是实际数据。
        Object x = awaitFulfill(s, e, timed, nanos);// 该方法作用就是，让当前线程等待。排除意外情况和超时的话，就是等待其他线程拿走数据并替换成 isData 不同的数据。
        if (x == s) { // x == s 是什么意思呢？ 表明在 awaitFulfill 方法中，这个数据被取消了，tryCancel 方法就是将 item 覆盖了 QNode。说明这次操作失败了。
            clean(t, s);// 操作失败则需要清理数据，并返回 null。
            return null;
        }

        // 如果一切顺利，确实被其他线程唤醒了，其他线程也交换了数据。
        // 这个判断：next != this，说明了什么？当这个 tail 节点的 next 不再指向自己，说明了
        if (!s.isOffList()) {           // not already unlinked
            // 这一步是将 S 节点设置为 Head，并且将新 Head 的 next 指向自己，让 Head 和之前的 next 断开。
            advanceHead(t, s);          // unlink if head     
            // 当 x 不是 null，表明对方线程是存放数据的。     
            if (x != null)              // and forget fields
                // 这一步操作将自己的 item 设置成自己。
                s.item = s;
            // 将 S 节点的持有线程变成 null。
            s.waiter = null;
        }
        // x 不是 null 表明，对方线程是生产者，返回他生产的数据；如果是 null，说明对方线程是消费者，那他自己就是生产者，返回自己的数据，表示成功。
        return (x != null) ? (E)x : e;

    } 
    // 2. 如果当前的操作类型和 tail 的操作不一样。称之为互补。
    else {                            // complementary-mode
        QNode m = h.next;               // node to fulfill
        // 如果下方这些判断没过，说明并发修改了，自旋重来。 
        if (t != tail || m == null || h != head)
            continue;                   // inconsistent read

        Object x = m.item;
        // 如果 head 节点的 isData 和当前操作相同，
        // 如果 操作不同，但 head 的 item 就是自身，也就是发生了取消操作，tryCancel 方法会做这件事情。
        // 如果上面2个都不满足，尝试使用 CAS 将 e 覆盖 item。 
        if (isData == (x != null) ||    // m already fulfilled
            x == m ||                   // m cancelled
            !m.casItem(x, e)) {         // lost CAS
            // CAS 失败了，Head 的操作类型和当前类型相同，item 被取消了，都会走这里。
            // 将 h.next 覆盖 head。重来。
            advanceHead(h, m);          // dequeue and retry
            continue;
        }
        // 这里也是将 h.next 覆盖 head。能够走到这里，说明，上面的 CAS 操作成功了，当前线程已经将 e 覆盖了 next 的 item 。
        advanceHead(h, m);              // successfully fulfilled
        // 唤醒 next 的 线程。提醒他可以取出数据，或者“我”已经拿到数据了。
        LockSupport.unpark(m.waiter);
        // 如果 x 不是 null，表明这是一次消费数据的操作，反之，这是一次生产数据的操作。
        return (x != null) ? (E)x : e;
    }
}
```

说实话，代码还是比较复杂的。JDK 中注释是这么说的：
> 基本算法是死循环采取 2 种方式中的其中一种。
   1 如果队列是空的，或者持有相同的模式节点（`isData` 相同），就尝试添加节点到队列中，并让当前线程等待。
  2 如果队列中有线程在等待，那么就使用一种`互补`的方式，使用 CAS 和等待者交换数据。并返回。
   

什么意思呢？

首先明确一点，队列中，数据有 2 种情况（但同时只存在一种），要么` QNode` 中有实际数据（`offer` 的时候，是有数据的，但没有“人”来取），要么没有实际数据（`poll` 的时候，队列中没有数据，线程只好等待）。队列在哪一种状态取决于`他为空后，第一个插入的是什么类型的数据`。

楼主画了点图来表示：

1. 队列初始化的时候，只有一个空的` Node`。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-9b6364f7cff9dc2c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 此时，一个线程尝试 `offer` 或者 `poll `数据，都会插入一个 `Node` 插入到节点中。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-6551db864a188878.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 假设刚刚发生的是 offer 操作，这个时候，另一个线程也来 offer，这时就会有 2 个节点。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-7fe5dc6026fb6b46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


4. 这个时候，队列中有 2 个有真实数据（offer 操作）的节点了，注意，这个时候，那 2 个线程都是 `wait `的，因为没有人接受他们的数据。此时，又来一个线程，做 poll 操作。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-70a7bfd773823bfc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上图可以看出，` poll` 线程从` head` 开始取数据，因为它的 `isData` 和 `tail` 节点的 isData 不同，那么就会从 head 开始找节点，并尝试将自己的 null 值和节点中的真实数据进行交换。并唤醒等待中的线程。

这 4 幅图就是 `SynchronousQueue `的精华。

既然叫做同步队列，一定是 A 线程生产数据的时候，有 B 线程在消费，否则 A  线程就需要等待，反之，如果 A 线程准备消费数据，但队列中没有数据，线程也会等待，直到有 B 线程存放数据。


而 JDK 的实现原理则是：使用一个队列，队列中的用一个 `isData` 来区分生产还是消费，所有新操作都根据 tail 节点的模式来决定到底是追加到 `tail `节点还是和 `tail `节点（从 `head` 开始）交换数据。

而所谓的交换是从` head `开始，取出节点的实际数据，然后使用 `CAS` 和匹配到的节点进行交换。从而完成两个线程直接交换数据的操作。

为什么他在某些情况下，比` LinkedBlockingQueue `性能高呢？其中有个原因就是没有使用锁，减少了线程上下文切换。第二则是线程之间交换数据的方式更加的高效。

  
好，重点部分讲完了，再看看其中线程是如何等待的。逻辑在 `awaitFulfill` 方法中：

```java
// 自旋或者等待，直到填充完毕
// 这里的策略是什么呢？如果自旋次数不够了，通常是 16 次，但还有超过 1 秒的时间，就阻塞等待被唤醒。
// 如果时间到了，就取消这次的入队行为。
// 返回的是 Node 本身
// s.item 就是 e 
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    Thread w = Thread.currentThread();
    int spins = ((head.next == s) ?// 如果成功将 tail.next 覆盖了 tail，如果有超时机制，则自旋 32 次，如果没有超时机制，则自旋 32 *16 = 512次
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        if (w.isInterrupted())// 当前线程被中断
            s.tryCancel(e);// 尝试取消这个 item
        Object x = s.item;// 获取到这个 tail 的 item
        if (x != e) // 如果不相等，说明 node 中的 item 取消了，返回这个 item。
            // 这里是唯一停止循环的地方。当 s.item 已经不是当初的哪个 e 了，说明要么是时间到了被取消了，要么是线程中断被取消了。
            // 当然，不仅仅只有这2种 “意外” 情况，还有一种情况是：当另一个线程拿走了这个数据，并修改了 item，也会通过这个判断，返回被“修改”过的 item。
            return x;
        if (timed) {// 如果有时间限制
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {// 如果时间到了
                s.tryCancel(e);// 尝试取消 item，供上面的 x != e 判断
                continue;// 重来
            }
        }
        if (spins > 0)// 如果还有自旋次数
            --spins;// 减一
        else if (s.waiter == null)// 如果自旋不够，且 tail 的等待线程还没有赋值
            s.waiter = w;// 当前线程赋值给 tail 的等待线程
        else if (!timed)// 如果自旋不够，且如果线程赋值过了，且没有限制时间，则 wait，（危险操作）
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)// 如果自旋不够，且如果限制了时间，且时间还剩余超过 1 秒，则 wait 剩余时间。
            // 主要目的就是等待，等待其他线程唤醒这个节点所在的线程。
            LockSupport.parkNanos(this, nanos);
    }
}
```

该方法逻辑如下：
1. 默认自旋 32 次，如果没有超时机制，则 512 次。
2. 如果时间到了，或者线程被中断，则取消这次的操作，将` item `设置成自己。供后面判断。
3. 如果自旋结束，且剩余时间还超过 1 秒，则阻塞等待至剩余时间。
4. 当线程被其他的线程唤醒，说明数据被交换了。则 `return`，返回的是交换后的数据。

## 总结

好了，关于 `SynchronousQueue `的核心源码分析就到这里了，楼主没有分析这个类的所有源码，只研究了核心部分代码，这足够我们理解这个 `Queue` 的内部实现了。

总结下来就是：

JDK 使用了队列或者栈来实现公平或非公平模型。其中，`isData` 属性极为重要，标识这这个线程的这次操作，决定了他到底应该是追加到队列中，还是从队列中交换数据。

每个线程在没有遇到自己的另一半时，要么快速失败，要么进行阻塞，阻塞等待自己的另一半来，至于对方是给数据还是取数据，取决于她自己，如果她是消费者，那么他就是生产者。

good luck！！！！








