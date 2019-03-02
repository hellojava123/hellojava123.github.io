---
layout: post
title: Netty-出站缓冲区-ChannelOutboundBuffer-源码解析（isWritable-属性的重要性）
date: 2018-03-19 11:11:11.000000000 +09:00
---
![](https://upload-images.jianshu.io/upload_images/4236553-e84d9611854eca36.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


目录：

## 前言

1. ChannelOutboundBuffer 介绍
2. addMessage 方法
3. addFlush 方法
4. flush0 方法
5. 缓冲区扩展思考
6. 总结

每个 ChannelSocket 的 Unsafe 都有一个绑定的  ChannelOutboundBuffer ，  Netty 向站外输出数据的过程统一通过 ChannelOutboundBuffer 类进行封装，目的是为了提高网络的吞吐量，在外面调用 write 的时候，数据并没有写到 Socket，而是写到了 ChannelOutboundBuffer 这里，当调用 flush 的时候，才真正的向 Socket 写出。同时，本文也关注当缓冲区满了的时候，Netty 如何处理。

## 1. ChannelOutboundBuffer 介绍

官方文档这么介绍的：
> (Transport implementors only) an internal data structure used by AbstractChannel to store its pending outbound write requests.
All methods must be called by a transport implementation from an I/O thread。
意思是，这个一个数据传输的实现者，一个内部的数据结构用于存储等待的出站写请求。所有的方法都必有由 IO 线程来调用。


既然该类有一个内部的数据结构，我们就看看他的数据结构的样子，有以下几个属性：

````java
private Entry flushedEntry; // 即将被消费的开始节点
private Entry unflushedEntry;// 被添加的开始节点，但没有准备好被消费。
private Entry tailEntry;// 最后一个节点
````


从上面的属性可以看出，这他么就是个链表。不过，这个链表有2个头，在调用 addFlush 方法的时候会将 unflushedEntry 赋值给 flushedEntry。表示即将从这里开始刷新。具体如下图：

![](https://upload-images.jianshu.io/upload_images/4236553-d55bf0096ae7dbd7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

调用 addMessage 方法的时候，创建一个 Entry ，将这个 Entry 追加到 TailEntry 节点后面，调用 addFlush 的时候，将 unflushedEntry 的引用赋给 flushedEntry，然后将 unflushedEntry 置为 null。

当数据被写进 Socket 后，从 flushedEntry（current） 节点开始，循环将每个节点删除。



关于这 3 个方法，我们后面详细解释。



## 2. addMessage 方法

该方法 doc 文档：
>Add given message to this ChannelOutboundBuffer. The given ChannelPromise will be notified once the message was written.
将给定的消息添加到 ChannelOutboundBuffer，一旦消息被写入，就会通知 promise。

代码如下：

````java
public void addMessage(Object msg, int size, ChannelPromise promise) {
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
        tailEntry = entry;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
        tailEntry = entry;
    }
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }
    incrementPendingOutboundBytes(entry.pendingSize, false);
}
````

说说方法步骤：
1. 根据 ByteBuf 相互属性和 promise 创建一个 Entry 节点。
2. 将新的节点追加到 tailEntry 节点上。如果考虑之前的全部被清空了话，则新节点就是唯一节点，unflushedEntry 属性就是新的节点。可对照上面的图来看。
3. 使用 CAS 将 totalPendingSize（总的数据大小） 属性增加 Entry 实例的大小（96 字节） + 真实数据的大小。


主要这个 Entry 节点的创建有点意思：


![](https://upload-images.jianshu.io/upload_images/4236553-59f7cbb7909ab494.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](https://upload-images.jianshu.io/upload_images/4236553-458cc28c298eddbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](https://upload-images.jianshu.io/upload_images/4236553-4f525db9ed4835de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Netty 将在 ThreadLocalMap 中存储了一个  Stack （栈）对象，存储重复使用的 DefaultHandle 实例，该实例的 value 属性就是 Entry ，所以这个 Entry 也是重复使用的，每次用完所有参数置为 null，再返回到栈中，下次再用，从这个栈中弹出。重复利用。对象池的最佳实践。而且是保存再线程中，速度更快，不会有线程竞争。这个设计倒是可以学习以下。


看完了 addMessage ，再看看 addFlush 方法。

## 3. addFlush 方法

当 addMessage 成功添加进 ChannelOutboundBuffer 后，就需要 flush 刷新到 Socket 中去。但是这个方法并不是做刷新到 Socket 的操作。而是将 unflushedEntry 的引用转移到 flushedEntry 引用中，表示即将刷新这个 flushedEntry，至于为什么这么做？
答：因为 Netty 提供了 promise，这个对象可以做取消操作，例如，不发送这个 ByteBuf 了，所以，在 write 之后，flush 之前需要告诉 promise 不能做取消操作了。

代码如下：

````java
public void addFlush() {
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            flushedEntry = entry;
        }
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);
        unflushedEntry = null;
    }
}
````


结合上面的图：
1. 首先拿到未刷新的头节点。
2. 判 null 之后，将这个 unflushedEntry 赋值给 flushedEntry，而这里的判 null 是做什么呢？防止多次调用 flush 。
3. 循环尝试设置这些节点，告诉他们不能做取消操作了，如果尝试失败了，就将这个节点取消，在调用 nioBuffers 方法的时候，这个节点会被忽略。同时将 totalPendingSize 相应的减小。

设置之后，promise 调用 cancel 方法就会返回 false。

在调用完 outboundBuffer.addFlush() 方法后，Channel 会调用 flush0 方法做真正的刷新。

## 4. flush0 方法

flush0 的核心是调用 dowrite 方法并传入 outboundBuffer。

![](https://upload-images.jianshu.io/upload_images/4236553-61d33ded018276b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


每种类型的 Channel 都实现都不一样。我们看的是 NioSocketChannel 的实现，方法很长，楼主截取重要逻辑：

````java
// 拿到NIO Socket
SocketChannel ch = javaChannel();
// 获取自旋的次数，默认16
int writeSpinCount = config().getWriteSpinCount();
// 获取设置的每个 ByteBuf 的最大字节数，这个数字来自操作系统的 so_sndbuf 定义
int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
// 调用 ChannelOutboundBuffer 的 nioBuffers 方法获取 ByteBuffer 数组，从flushedEntry开始，循环获取
ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
// ByteBuffer 的数量
int nioBufferCnt = in.nioBufferCount();
// 使用 NIO 写入 Socket 
ch.write(buffer);
// 调整最大字节数
adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
// 删除 ChannelOutboundBuffer 中的 Entry
in.removeBytes(localWrittenBytes);
// 自旋减一，直到自旋小于0停止循环，当然如果 ChannelOutboundBuffer 空了，也会停止。
--writeSpinCount;
// 如果自旋16次还没有完成 flush，则创建一个任务放进mpsc 队列中执行。
incompleteWrite(writeSpinCount < 0);
````

上面的注释基本就是 flush 的逻辑。

* **当然 flush0 方法在 NIO 的具体实现中，还加入了对注册事件的判断：**

````java
protected final void flush0() {
    if (!isFlushPending()) {
        super.flush0();
    }
}


private boolean isFlushPending() {
    SelectionKey selectionKey = selectionKey();
    return selectionKey.isValid() && (selectionKey.interestOps() & SelectionKey.OP_WRITE) != 0;
}
````

这里的判断是：如果注册了写事件，就暂时不写了，因为缓冲区到了水位线了，所以这次直接返回，等会再写。等到 EventLoop 触发写事件了，就会调用  `ch.unsafe().forceFlush()` 方法将数据刷新到 TCP 缓冲区。

- **这里有一个小知识点：**
> NIO 的写事件大部分时候是不需要注册的，只有当 TCP 缓冲区达到水位线了，不能写入了，才需要注册写事件。当缓冲区有空间了，NIO 就会触发写事件。



## 5. 缓冲区扩展思考

从上面的逻辑上来看，不直到大家有没有发现一个问题：如果对方 Socket 接收很慢，ChannelOutboundBuffer 就会积累很多的数据。并且这个 ChannelOutboundBuffer  是没有大小限制的链表。可能会导致 OOM，Netty 已经考虑了这个问题，在　addMessage　方法的最后一行，incrementPendingOutboundBytes方法，会判断　totalPendingSize　的大小是否超过了高水位阈值（默认64 kb），如果超过，关闭写开关，调用 piepeline  的 fireChannelWritabilityChanged 方法可改变 flush 策略。


![](https://upload-images.jianshu.io/upload_images/4236553-0578a3188c6f5848.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4236553-a97a3af8330cd752.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4236553-2e9f3a4695acde09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4236553-b8c3bf159102b5b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4236553-1fb36d876b4f0925.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于 channelWritabilityChanged API，Netty 这样解释：
> 当 Channel 的可写状态发生改变时被调用。用户可以确保写操作不会完成的太快（以避免发生 OOM）或者可以在 Channel 变为再次可写时恢复写入。可以通过调用 Channel 的 isWritable 方法来检测 Channel 的可写性。与可写性相关的阈值可以通过 Channel.config().setWriteBufferHighWaterMark 和 Channel.config().setWriteBufferLowWaterMark 方法来设置，默认最小 32 kb，最大 64 kb。



那么，上面时候恢复可写状态呢？remove 的时候，或者 addFlush 是丢弃了某个节点，会对 totalPendingSize 进行削减，削减之后进行判断。如果 totalPendingSize 小于最低水位了。就恢复写入。


![](https://upload-images.jianshu.io/upload_images/4236553-9a486d98e28f5bfb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**也就是说，默认的情况下，ChannelOutboundBuffer  缓存区的大小最大是 64 kb，最小是 32 kb，哪里看出来的呢？**

![](https://upload-images.jianshu.io/upload_images/4236553-19cbea156a36c817.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然了，可以在 option 选项中进行修改，API 文档也说过了。

当不能写的时候，就会调用 ChannelWritabilityChanged 方法，用户可以在代码中，让写操作进行的慢一点。



## 6. 总结

到了总结的时刻。

Netty 的 write 的操作不会立即写入，而是存储在了 ChannelOutboundBuffer 缓冲区里，这个缓冲区内部是 Entry 节点组成的链表结构，通过 addMessage 方法添加进链表，通过 addFlush 方法表示可以开始写入了，最后通过 SocketChannel 的 flush0 方法真正的写入到 JDK 的 Socket 中。同时需要注意如果 TCP 缓冲区到达一个水位线了，不能写入 TCP 缓冲区了，就需要晚点写入，这里的方法判断是 isFlushPending（）。

其中，有一个需要注意的点就是，如果对方接收数据较慢，可能导致缓冲区存在大量的数据无法释放，导致OOM，Netty 通过一个 isWritable 开关尝试解决此问题，但用户需要重写 ChannelWritabilityChanged 方法，因为一旦超过默认的高水位阈值，Netty 就会调用 ChannelWritabilityChanged 方法，执行完毕后，继续进行 flush。用户可以在该方法中尝试慢一点的操作。等到缓冲区的数据小于低水位的值时，开关就关闭了，就不会调用 ChannelWritabilityChanged 方法。因此，合理设置这两个数值也挺重要的。

好，限于篇幅，关于 ChannelOutboundBuffer  的分析就到这里，今天说的这几个方法算是这个类的主要方法，因为 Netty 的写操作都是围绕这三个方法来的。


good luck！！！！！








