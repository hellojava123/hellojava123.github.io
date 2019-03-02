---
layout: post
title: 并发编程之-Exchanger-源码分析
date: 2018-04-15 11:11:11.000000000 +09:00
---
## 前言

JUC 包中除了 CountDownLatch, CyclicBarrier, Semaphore, 还有一个重要的工具,只不过相对而言使用的不多,什么呢? Exchange —— 交换器。用于在两个线程之间交换数据，A 线程将 a 数据交给 B 线程，B 线程将 b 数据交给 a 线程。

具体使用例子参见 [并发编程之 线程协作工具类](http://thinkinjava.cn/article/37)。我们这篇文章就不再讲述如何使用了。

而今天，我们将从源码处分析，Exchange 的实现原理。如果大家看过之前关于 SynchronousQueue 的文章  [并发编程之 SynchronousQueue 核心源码分析](http://thinkinjava.cn/article/95)，就能够看的出来，Exchange 的原理和他很类似。

## 1. 源码

类 UML：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-945d4f8ddf29ce92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

内部有 2 个内部类： Node ， Participant 重写了 ThreadLocal 的 initialValue 方法。

构造方法如下：

```java
public Exchanger() {
    participant = new Participant();
}

static final class Participant extends ThreadLocal<Node> {
    public Node initialValue() { return new Node(); }
}
```

就是创建了一个 ThreadLocal 对象，并设置了初始值，一个 Node 对象。

看看这个 node 对象：

```java
@sun.misc.Contended 
static final class Node {
    int index;              //  node 在 arena 数组下标
    int bound;              //  交换器的最后记录值 
    int collides;           //  记录的 CAS 失败数
    int hash;               //  伪随机的自旋数
    Object item;            //  这个线程的数据项
    volatile Object match;  //  别的线程提供的元素,也就是释放他的线程提供的数据 item
    volatile Thread parked; //  当阻塞时，设置此线程，不阻塞的话就不必了(因为会自旋)
}
```

这个 node 对象就是 A ，B 线程实际存储数据的容器。A 线程存在 item 属性上，B 线程存储在 match 线程上，称为匹配。同时，有个线程对象，你应该猜到做什么用处的吧，对，挂起线程的。

和 SynchronousQueue 的区别在于， SynchronousQueue 使用了一个变量来保存数据项，通过 isData 来区别 “存” 操作和 “取” 操作。而 Exchange 使用了 2 个变量，就不用使用 isData 来区分了。


我们再来看看 Exchange 的唯一重要方法 ： exchange 方法。

## 2. exchange 方法源码分析

代码如下：

```java
public V exchange(V x) throws InterruptedException {
    Object v;
    Object item = (x == null) ? NULL_ITEM : x; // translate null args
    // arena 不是 Null ,返回的却是 null, 说明线程中断了.
    // 如果 arena 是 null, 就执行后面的方法.反之,如果不是 null, 执行没有意义.
    // 注意,当 slotExchange 有机会被执行,且返回的不是 null, 这个表达式整个就是 false, 下面的表达式就不会执行了.
    // 也就是说,当 slot 有效的时候, arena 是没有必要执行的.
    if ((arena != null || (v = slotExchange(item, false, 0L)) == null) &&
        // 线程中断了,或者返回的是 null. 说明线程中断了
        // 如果线程没有中断 ,就执行后面的方法.
        ((Thread.interrupted() || (v = arenaExchange(item, false, 0L)) == null))){
        throw new InterruptedException();        
    }
    return (v == NULL_ITEM) ? null : (V)v;
}
```

说一下方法的逻辑：
1. 如果执行 slotExchange 有结果,就不再执行 arenaExchange.
2. 如果 slot 被占用了,就执行 arenaExchange.

返回值是什么呢？返回值就是对方线程的数据项，如果 A 线程先调用，那么 A 线程将数据项存在 item 中，B 线程后调用，则 B 线程将数据存在 match 属性中。

A 返回的是 match 属性，b 返回的是 item 属性。


从该方法中，可以看到，有 2 个重要的方法： slotExchange， arenaExchange。先简单说说这两个方法。

当没有多线程并发操作 Exchange 的时候，使用 slotExchange 就足够了。 slot 是一个 node 对象。

当出现并发了，一个 slot 就不够了，就需要使用一个 node 数组 arena 操作了。

so，我们先看看 slotExchange 方法吧，两个方法的逻辑类似。


## 3. slotExchange 方法源码分析

代码加注释如下：

```java
 private final Object slotExchange(Object item, boolean timed, long ns) {
        Node p = participant.get(); // 从 ThreadLocal 中取出 node 对象
        Thread t = Thread.currentThread();// 当前线程
        if (t.isInterrupted()) // preserve interrupt status so caller can recheck
            return null;

        for (Node q;;) {// 死循环
            // 另一个下线程进入这里, 假设 slot 有值
            if ((q = slot) != null) {
                // 将  slot 修改为 null
                if (U.compareAndSwapObject(this, SLOT, q, null)) {
                    // 拿到 q 的 item
                    Object v = q.item;
                    // 自己的 item 赋值给 match,以让对方线程获取
                    q.match = item;
                    // q 线程
                    Thread w = q.parked;
                    // slot 的  parked 就是阻塞等待的线程对象.
                    if (w != null)
                        U.unpark(w);
                    // 返回了上一个线程放入的 item
                    return v;
                }
                // 如果使用 CAS 修改slot 失败了,说明 slot 被使用了,那就需要创建 arena 数组了
                if (NCPU > 1 && bound == 0 &&
                    U.compareAndSwapInt(this, BOUND, 0, SEQ)) // SEQ == 256; 默认 BOUND == 0
                    arena = new Node[(FULL + 2) << ASHIFT];// length = (2 + 2) << 7 == 512
            }
            // 如果 slot 是 null, 但 arena 有值了,说明有线程竞争 slot 了,返回 null, 执行 arenaExchange 逻辑
            else if (arena != null)
                return null; // caller must reroute to arenaExchange
            else {// 第一次循环,给 p node 的 item 赋值
                p.item = item;
                // 将 slot 赋值赋值为 p
                if (U.compareAndSwapObject(this, SLOT, null, p))
                    // 赋值成功跳出循环
                    break;
                // 如果 CAS 失败,将 p 的值清空,重来
                p.item = null;
            }
        }
        // 当走到这里的时候,说明 slot 是 null, 且 arena 不是 null(没有多线程竞争使用 slot),并且成功将 item 放入了 slot 中.
        // 这个时候要做的就是阻塞自己,等待对方取出 slot 的数据项,然后重置 slot 的数据和池化对象的数据
        // 伪随机数
        int h = p.hash;
        // 超时时间 
        long end = timed ? System.nanoTime() + ns : 0L;
        // 自旋,默认 1024
        int spins = (NCPU > 1) ? SPINS : 1;
        Object v;
        // 如果这个值不是 null, 说明数据被其他线程拿走了, 并且其他线程将数据赋值给 match 属性,完成了一次交换
        while ((v = p.match) == null) {
            // 自旋
            if (spins > 0) {
                // 计算伪随机数
                h ^= h << 1; h ^= h >>> 3; h ^= h << 10;
                // 如果算出来的是0,就使用线程 ID
                if (h == 0)
                    h = SPINS | (int)t.getId();
                // 如果不是0,就将自旋数减一,并且让出 CPU 时间片
                else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0)
                    Thread.yield();
            }
            // 如果自旋数不够了,且 slot 还没有得到,就重置自旋数
            else if (slot != p)
                spins = SPINS;
            // 如果 slot == p 了,说明对 slot 赋值成功
            // 如果线程没有中断 && 数组不是 null && 没有超时限制
            else if (!t.isInterrupted() && arena == null &&
                     (!timed || (ns = end - System.nanoTime()) > 0L)) {
                // 为线程中的 parkBlocker 属性赋值为 Exchange 自己
                U.putObject(t, BLOCKER, this);
                // node 节点的阻塞线程为当前线程
                p.parked = t;
                // 如果这个数据还没有被拿走,阻塞自己
                if (slot == p)
                    U.park(false, ns);
                // 线程苏醒后,将 p 的阻塞线程属性清空
                p.parked = null;
                // 将当前线程的 parkBlocker 属性设置成 null
                U.putObject(t, BLOCKER, null);
            }
            // 如果有超时限制,使用 CAS 将 slot 从 p 变成 null,取消这次交换
            else if (U.compareAndSwapObject(this, SLOT, p, null)) {
                // 如果CAS成功,如果时间到了 && 线程没有中断 : 返回 time_out 对象: 返回 null
                v = timed && ns <= 0L && !t.isInterrupted() ? TIMED_OUT : null;
                // 跳出内层循环
                break;
            }
        }
        // 将 p 的 match 属性设置成 null, 表示初始化状态,没有任何匹配  >>>  putOrderedObject是putObjectVolatile的内存非立即可见版本.
        U.putOrderedObject(p, MATCH, null);
        // 重置 item
        p.item = null;
        // 保留伪随机数,供下次种子数字
        p.hash = h;
        // 返回
        return v;
    }
```

源码还是有点小长的。简单说说逻辑。

Exchange 使用了对象池的技术,将对象保存在 ThreadLocal 中,这个对象(Node)封装了数据项,线程对象等关键数据。

当第一个线程进入的时候,会将数据放到 池化对象中,并赋值给 slot 的 item.并阻塞自己(通常不会立即阻塞,而是使用 yield 自旋一会儿),等待对方取值.

当第二个线程进入的时候,会拿出存储在 slot item 中的值, 然后对 slot 的 match 赋值,并唤醒上次阻塞的线程.

当第一个线程阻塞被唤醒后,说明对方取到值了,就获取 slot 的 match 值, 并重置 slot 的数据和池化对象的数据,并返回自己的数据.

如果超时了,就返回 Time_out 对象.

如果线程中断了,就返回 null.

在该方法中,会返回 2 种结果,一是有效的 item, 二是 null--- 要么是线程竞争使用 slot 了,创建了 arena 数组,要么是线程中断了.

用一幅图来看看具体逻辑，其实还是挺简单的。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-fcb6dc8a5cf7412b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当 slot 被别是线程使用了，那么就需要创建一个 arena 的数组了。通过操纵数组里面的元素来实现数据交换。

关于 arenaExchange 方法的源码我就不贴了，有 2 个原因，一个是总体逻辑和 slotExchange 相同，第二个原因则是，其中有一些细节我没有弄懂，就不发出自己写代码注释了，防止误导。但我们已经掌握了 Exchange 的原理。


## 总结

Exchange 和 SynchronousQueue 类似，都是通过两个线程操作同一个对象实现数据交换，只不过就像我们开始说的，SynchronousQueue 使用的是同一个属性，通过不同的 isData 来区分，多线程并发时，使用了队列进行排队。

Exchange 使用了一个对象里的两个属性，item 和 match，就不需要 isData 属性了，因为在 Exchange 里面，没有 isData 这个语义。而多线程并发时，使用数组来控制，每个线程访问数组中不同的槽。

最后，用我们的图收尾吧：


![image.png](https://upload-images.jianshu.io/upload_images/4236553-405970f00b740114.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)







