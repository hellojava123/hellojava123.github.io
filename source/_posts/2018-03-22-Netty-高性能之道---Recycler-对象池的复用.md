---
layout: post
title: Netty-高性能之道---Recycler-对象池的复用
date: 2018-03-22 11:11:11.000000000 +09:00
---
![Recycler 设计图](https://upload-images.jianshu.io/upload_images/4236553-552d2878c402b15c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言 

我们知道，Java 创建一个实例的消耗是不小的，如果没有使用栈上分配和 TLAB，那么就需要使用 CAS 在堆中创建对象。所以现在很多框架都使用对象池。Netty 也不例外，通过重用对象，能够避免频繁创建对象和销毁对象带来的损耗。

来看看具体实现。


## 1. Recycler 抽象类简介

该类 doc：

>  Light-weight object pool based on a thread-local stack.
基于线程局部堆栈的轻量级对象池。

该类是个容器，内部主要是一个 Stack 结构。当需要使用一个实例时，就弹出，当使用完毕时，就清空后入栈。

* 该类有 2 个主要方法：

````java
1. public final T get() // 从 threadLocal 中取出 Stack 中首个 T 实例。
2. protected abstract T newObject(Handle<T> handle) // 当 Stack 中没有实例的时候，创建一个实例返回。
````

* 该类有 4 个内部接口 / 内部类：
````java
// 定义 handler 回收实例
public interface Handle<T> {
    void recycle(T object);
}

// Handle 的默认实现，可以将实例回收，放入 stack。
static final class DefaultHandle<T> implements Handle<T>

// 存储对象的数据结构。对象池的真正的 “池”
static final class Stack<T>

// 多线程共享的队列
private static final class WeakOrderQueue

// 队列中的链表结构，用于存储多线程回收的实例
private static final class Link extends AtomicInteger
````


* 实现线程局部缓存的 FastThreadLocal：
````java
private final FastThreadLocal<Stack<T>> threadLocal = new FastThreadLocal<Stack<T>>() {
    @Override
    protected Stack<T> initialValue() {
        return new Stack<T>(Recycler.this, Thread.currentThread(), maxCapacityPerThread, maxSharedCapacityFactor,
                ratioMask, maxDelayedQueuesPerThread);
    }

    @Override
    protected void onRemoval(Stack<T> value) {
        if (value.threadRef.get() == Thread.currentThread()) {
           if (DELAYED_RECYCLED.isSet()) {
               DELAYED_RECYCLED.get().remove(value);
           }
        }
    }
};
````

* 核心方法 get 操作

````java
public final T get() {
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE);
    }
    Stack<T> stack = threadLocal.get();
    DefaultHandle<T> handle = stack.pop();
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle);
    }
    return (T) handle.value;
}
````


* 核心方法 DefaultHandle 的 recycle 操作
````java
public void recycle(Object object) {
    if (object != value) {
        throw new IllegalArgumentException("object does not belong to handle");
    }
    stack.push(this);
}
````


*****

## 2. Netty 中的使用范例

> **io.netty.channel.ChannelOutboundBuffer.Entry 类**

* 示例代码如下：

````java
// 实现了 Recycler 抽象类
private static final Recycler<Entry> RECYCLER = new Recycler<Entry>() {
    protected Entry newObject(Handle<Entry> handle) {
        return new Entry(handle);
    }
};

// 创建实例
Entry entry = RECYCLER.get();
// doSomeing......
// 归还实例
entry.recycle();
````

从上面的 get 方法，我们知道，最终会从 threadLocal 取出 Stack，从 Stack 中弹出 DefaultHandle 对象（如果没有就创建一个），然后调用我们重写的 newObject 方法，将创建的对象和 handle 绑定。最后返回这个对象。

当调用 entry.recycle() 方法的时候，实际会调用 DefaultHandle 的 recycle 方法。我们看看该方法实现：

````java
public void recycle(Object object) {
    if (object != value) {
        throw new IllegalArgumentException("object does not belong to handle");
    }
    stack.push(this);
}
````

这里的 value 就是 get 方法中赋值的。如果不相等，就抛出异常。反之，将 handle 入栈 stack。注意：这里并没有对 value 做任何处理，只是在 Entry 内部做了清空处理。所以，这个 handle 和 handle 绑定的对象就保存在了 stack 中。

下次再次调用 get 时，就可以直接从该 threadLocal 中取出 handle 和 handle 绑定的 value了。完成了一次完美的对象池的实践。也就是说，一个 handle 绑定一个实例。而这个 handle 还是比较轻量的。

从这里可以看出，Stack 就是真正的 “池子”。我们就看看这个池子的内部实现。

而这个 stack 对外常用的方法的 pop 和 push。我们就来看看这两个方法。

##  3. pop 方法

代码如下：

````java
DefaultHandle<T> pop() {
    int size = this.size;
    if (size == 0) {
        if (!scavenge()) {
            return null;
        }
        size = this.size;
    }
    size --;
    DefaultHandle ret = elements[size];
    elements[size] = null;
    if (ret.lastRecycledId != ret.recycleId) {
        throw new IllegalStateException("recycled multiple times");
    }
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    this.size = size;
    return ret;
}
````
逻辑如下：
1. 拿到这个 Stack 的长度，实际上，这个 Stack 就是一个 DefaultHandle 数组。
2. 如果这个长度是 0，没有元素了，就调用 scavenge 方法尝试从 queue 中转移一些数据到 stack 中。scavenge 方法待会详细再讲。
3. 重置 size 属性和其余两个属性。返回实例。

这个方法除了 scavenge 之外，还是比较简单的。



## 4. push 方法

代码如下：

````java
 void push(DefaultHandle<?> item) {
    Thread currentThread = Thread.currentThread();
    if (threadRef.get() == currentThread) { 
        pushNow(item);
    } else { 
        pushLater(item, currentThread);
    }
}
````

当一个对象使用 pop 方法取出来之后，可能会被别的线程使用，这时候，如果是你，你怎么处理呢？

先看看当前线程的处理：

看看 pushNow 方法：

````java
private void pushNow(DefaultHandle<?> item) {
    if ((item.recycleId | item.lastRecycledId) != 0) {
        throw new IllegalStateException("recycled already");
    }
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;

    int size = this.size;
    if (size >= maxCapacity || dropHandle(item)) {
        return;
    }
    if (size == elements.length) {
        elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
    }

    elements[size] = item;
    this.size = size + 1;
}
````
该方法主要逻辑如下：
1. 如果 Stack 大小已经大于等于最大容量或者这个 handle 在容器里了，就不做回收了。
2. 如果数组满了，扩容一倍，最大 4096（默认）。
3. size + 1。

看看 dropHandle 方法的实现：

````java
boolean dropHandle(DefaultHandle<?> handle) {
    // 没有被回收过
    if (!handle.hasBeenRecycled) {
        // 第一次是 -1，++ 之后变为0，取余7。其实如果正常情况下，结果应该都是0。
        // 如果下面的判断不是0 的话，那么已经归还。这个对象就没有必要重复归还。
        // 直接丢弃。
        if ((++handleRecycleCount & ratioMask) != 0) {
            // Drop the object.
            return true;
        }
        // 改为被回收过，下次就不会进入了
        handle.hasBeenRecycled = true;
    }
    // 删除失败
    return false;
}
````

已经写了注释，就不再过多解释。


可以看到，pushNow 方法还是很简单的。由于在当前线程里，只需要还原到 Stack 的数组中就好了。


**关键是：如果是其他的线程做回收操作，该怎么办？**


## 5. pushLater 方法（多线程回收如何操作）

先说说 Netty 的解决办法和思路：

* 每个线程都有一个 Stack 对象，每个线程也都有一个软引用 Map，键为 Stack，值是 queue。

* 线程每次从 local 中获取 Stack 对象，再从 Stack 中取出实例。如果取不到，尝试从 queue 取，也就是从queue 中的 Link 中取出，并销毁 Link。

* 但回收的时候，可能就不是原来的那个线程了，由于回收时使用的还是原来的 Stack，所以，需要考虑这个实例如何才能正确的回收。

* 这个时候，就需要 Map 出场了。创建一个 queue 关联这个 Stack，将数据放到这个 queue 中。等到持有这个 Stack 的线程想拿数据了，就从 Stack 对应的 queue 中取出。

* 看出来了吗？只有一个线程持有唯一的 Stack，其余的线程只持有这个 Stack 关联的 queue。因此，可以说，这个 queue 是两个线程共享的。除了 Stack 自己的线程外，其余的线程的归还都是放到 自己的queue 中。

* 这个 queue 是无界的。内部的 Link 是有界的。每个线程对应一个 queue。

* 这些线程的 queue 组成了链表。


具体如下图所示：

![Recycler 设计图](https://upload-images.jianshu.io/upload_images/4236553-552d2878c402b15c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



看完了设计，再看看代码实现：

**pushLater 方法**
````java
private void pushLater(DefaultHandle<?> item, Thread thread) {
    // 每个 Stack 对应一串 queue，找到当前线程的 map
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    // 查看当前线程中是否含有这个 Stack 对应的队列
    WeakOrderQueue queue = delayedRecycled.get(this);
    if (queue == null) {// 如果没有
        // 如果 map 长度已经大于最大延迟数了，则向 map 中添加一个假的队列
        if (delayedRecycled.size() >= maxDelayedQueues) {// 8
            delayedRecycled.put(this, WeakOrderQueue.DUMMY);
            return;
        }
        // 如果长度不大于最大延迟数，则尝试创建一个queue，链接到这个 Stack 的 head 节点前（内部创建Link）
        if ((queue = WeakOrderQueue.allocate(this, thread)) == null) {
            // drop object
            return;
        }
        delayedRecycled.put(this, queue);
    } else if (queue == WeakOrderQueue.DUMMY) {
        // drop object
        return;
    }

    queue.add(item);
}    
````

该方法步骤如下：
1. 从 threadLcoal 中取出当前线程的 Map，尝试从 Map 中取出 Stack 映射的 queue。
2. 如果没有，就调用 WeakOrderQueue.allocate(this, thread) 方法创建一个。然后将这个 Stack 和 queue 绑定。
3. 将实例添加到这个 queue 中。

**我们主要关注如何 allocate 方法，关键方法 newQueue：**

````java
@1 
static WeakOrderQueue newQueue(Stack<?> stack, Thread thread) {
    WeakOrderQueue queue = new WeakOrderQueue(stack, thread);
    stack.setHead(queue);
    return queue;
}

@2 
private WeakOrderQueue(Stack<?> stack, Thread thread) {
    head = tail = new Link();
    owner = new WeakReference<Thread>(thread);
    availableSharedCapacity = stack.availableSharedCapacity;
}

@3
private static final class Link extends AtomicInteger {
    private final DefaultHandle<?>[] elements = new DefaultHandle[LINK_CAPACITY];
    private int readIndex;
    private Link next;
}

@4
synchronized void setHead(WeakOrderQueue queue) {
    queue.setNext(head);
    head = queue;
}
````

代码1，2，3，4。
1. 调用  WeakOrderQueue 构造方法，传入 stack 和 thread。
2. 创建一个 Link 对象，赋值给链表中的 head 和 tail。
3. Lind 的构造函数，也是一个链表。其中包含了保存实例的 Handle 数组，默认 16.
4. 将这个新的 queue 设置为该 stack 的 head 节点。

其中，有一个需要注意的地方就是   owner = new WeakReference<Thread>(thread)，使用了弱引用，当这个线程对象被 GC 后，这个 owner 也会变为 null，就可以像 threadLoca 一样对该引用进行判 null，来检查这个线程对象是否回收了。


再看看如何添加进 queue 中的：

````java
void add(DefaultHandle<?> handle) {
    handle.lastRecycledId = id;
    Link tail = this.tail;
    int writeIndex;
    if ((writeIndex = tail.get()) == LINK_CAPACITY) {
        if (!reserveSpace(availableSharedCapacity, LINK_CAPACITY)) {
            // Drop it.
            return;
        }
        this.tail = tail = tail.next = new Link();
        writeIndex = tail.get();
    }
    tail.elements[writeIndex] = handle;
    handle.stack = null;
    tail.lazySet(writeIndex + 1);
}
````

首先，拿到这个 queue 的 tail 节点，如果这个 tiail 节点满了，查看是否还有共享空间，如果没了，就丢弃这个实例。
反之，则新建一个 Link，追加到 tail 节点的尾部。然后，将数据插入新 tail 的数组。然后，将这个 handle 的 stack 属性设置成 null，表示这个 handle 不属于任何 statck 了，其他 stack 都可以使用。


**数据放进去了，怎么取出来呢？**

## 6. scavenge 方法

我们刚刚留了这个方法，现在可以开始讲了。代码如下：

````java
boolean scavenge() {
    // continue an existing scavenge, if any
    // 清理成功后，stack 的 size 会变化
    if (scavengeSome()) {
        return true;
    }

    // reset our scavenge cursor
    prev = null;
    cursor = head;
    return false;
}
````

主要调用的是 scavengeSome 方法，返回 true 表示将 queue 中的数据转移成功。看看该方法。

````java
boolean scavengeSome() {
    WeakOrderQueue prev;
    WeakOrderQueue cursor = this.cursor;
    if (cursor == null) {
        prev = null;
        cursor = head;
        if (cursor == null) {
            return false;
        }
    } else {
        prev = this.prev;
    }
    boolean success = false;
    do {
        // 将 head queue 的实例转移到 this stack 中
        if (cursor.transfer(this)) {
            success = true;
            break;
        }
        // 如果上面失败，找下一个节点
        WeakOrderQueue next = cursor.next;
        // 如果当前线程被回收了，
        if (cursor.owner.get() == null) {
          // 只要最后一个节点还有数据，就一直转移
            if (cursor.hasFinalData()) {
                for (;;) {
                    if (cursor.transfer(this)) {
                        success = true;
                    } else {
                        break;
                    }
                }
            }
            if (prev != null) {
                prev.setNext(next);
            }
        } else {
            prev = cursor;
        }
        cursor = next;
    } while (cursor != null && !success);
    // 转移成功之后，将 cursor 重置
    this.prev = prev;
    this.cursor = cursor;
    return success;
}
````

方法还是挺长的。我们拆解一下：
1. 拿到这个 stack 的 queue，调用这个 queue 的 transfer 方法，如果成功，结束循环。
2. 如果 queue 所在的线程被回收了，就将这个线程对应的 queue 中的所有数据全部转移到 stack 中。

可以看到，最重要的还是 transfer 方法。然而该方法更长，就不贴代码了，说说主要逻辑，有兴趣可以自己看看，逻辑如下：
1. 拿到这个 queue 的 head 节点，也就是 Link。如果 head 是 null，取出 next。
2. 循环 Link 中的实例，将其赋值到 stack 数组中。并将刚刚 handle 置为 null  的 stack 属性赋值。
3. 最后，将 statck 的 size 属性更新。

**其中有一个疑问：为什么在其他线程插入 Link 时将 handle 的 stack 的属性置为 null？在取出时，又将 handle 的 stack 属性恢复。**

答：因为如果 stack 被用户手动置为 null，而容器中的 handle 还持有他的引用的话，就无法回收了。同时 Map 也使用了软引用map，当 stack 没有了引用被 GC 回收时，对应的 queue 也就被回收了。避免了内存泄漏。实际上，在之前的 Recycler 版本中，确实存在内存泄漏的情况。


该方法的主要目的就是将 queue 所属的 Link 中的数据转移到 stack 中。从而完成多线程的最终回收。

## 总结

Netty 并没有使用第三方库实现对象池，而是自己实现了一个相对轻量的对象池。通过使用 threadLocal，避免了多线程下取数据时可能出现的线程安全问题，同时，为了实现多线程回收同一个实例，让每个线程对应一个队列，队列链接在 Stack 对象上形成链表，这样，就解决了多线程回收时的安全问题。同时，使用了软引用的map 和 软引用的 thradl 也避免了内存泄漏。

在本次的源码阅读中，确实收获很大。再回顾以下 Recycler 的设计图吧。设计的真的非常好。



![Recycler 设计图](https://upload-images.jianshu.io/upload_images/4236553-0594c2f96c319a8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



















































































