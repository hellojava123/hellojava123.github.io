---
layout: post
title: Netty-源码剖析之-unSafe-write-方法
date: 2018-03-19 11:11:11.000000000 +09:00
---
![](https://upload-images.jianshu.io/upload_images/4236553-2c336d4aeaea55b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 前言

在  [Netty 源码剖析之 unSafe.read 方法](https://www.jianshu.com/p/bb579eb4fadd) 一文中，我们研究了 read 方法的实现，这是读取内容到容器，再看看 Netty 是如何将内容从容器输出 Channel 的吧。


## 1. ctx.writeAndFlush 方法

当我们调用此方法时，会从当前节点找上一个 outbound 节点，进行，并调用下个节点的 write 方法。具体看代码：

````java
@1
public ChannelFuture writeAndFlush(Object msg) {
    return writeAndFlush(msg, newPromise()); // 创建了一个默认的 DefaultChannelPromise 实例，返回的就是这个实例。

@2
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    if (isNotValidPromise(promise, true)) {//判断 promise 有效性
        ReferenceCountUtil.release(msg);// 释放内存
        return promise;
    }

    write(msg, true, promise);

    return promise;
}

@3
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}
````


最终调用的就是 @3 方法。找到上一个 outbound 节点，判断他的节点是否时当前线程。如果是，则会直接调用，泛着，将后面的工作封装成一个任务放进 mpsc 队列，供当前线程稍后执行。这个 任务的 run 方法如下：

````java
public final void run() {
    try {
        if (ESTIMATE_TASK_SIZE_ON_SUBMIT) {
            ctx.pipeline.decrementPendingOutboundBytes(size);
        }
        write(ctx, msg, promise);
    } finally {
        ctx = null;
        msg = null;
        promise = null;
        handle.recycle(this);
    }
}

protected void write(AbstractChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
    ctx.invokeWrite(msg, promise);
}
````

最终执行的是 invokeWrite 方法。


我们看看如果直接执行会如何处理。

````java
private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
        invokeFlush0();
    } else {
        writeAndFlush(msg, promise);
    }
}
````

先执行 invokeWrite0 方法进行 write，然后 flush。

![@1](https://upload-images.jianshu.io/upload_images/4236553-e09ae6d795112fa9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![@2](https://upload-images.jianshu.io/upload_images/4236553-f0c987de5b04d088.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最终执行的是 unSafe 的 write 方法。**注意：当前节点已经到了 Head 节点。**

详细说说该方法。

## 2. unSafe  的 write 方法

````java
public final void write(Object msg, ChannelPromise promise) {
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
        msg = filterOutboundMessage(msg);
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }

    outboundBuffer.addMessage(msg, size, promise);
}
````


方法步骤如下：
1. 判断 outboundBuffer 有效性。
2. 将 ByteBuf 过滤成池化或者线程局部直接内存（如果不是直接内存的话）。
3. 预估当前 ByteBuf 大小（就是可读字节数）。
4. 将 ByteBuf 包装成一个 Entry 节点放入到 outboundBuffer 的单向链表中。

这里有一个地方需要注意一下， filterOutboundMessage 方法。

![](https://upload-images.jianshu.io/upload_images/4236553-ee8f5fbb83da5912.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果是直接内存的话，就直接返回了 ，反之调用 newDirectBuffer 方法。我们猜想肯定是重新包装成直接内存，利用直接内存令拷贝的特性，提升性能。

看看该方法内部逻辑：

````java
protected final ByteBuf newDirectBuffer(ByteBuf buf) {
    final int readableBytes = buf.readableBytes();
    if (readableBytes == 0) {
        ReferenceCountUtil.safeRelease(buf);
        return Unpooled.EMPTY_BUFFER;
    }

    final ByteBufAllocator alloc = alloc();
    if (alloc.isDirectBufferPooled()) {
        ByteBuf directBuf = alloc.directBuffer(readableBytes);
        directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
        ReferenceCountUtil.safeRelease(buf);
        return directBuf;
    }

    final ByteBuf directBuf = ByteBufUtil.threadLocalDirectBuffer();
    if (directBuf != null) {
        directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
        ReferenceCountUtil.safeRelease(buf);
        return directBuf;
    }

    // Allocating and deallocating an unpooled direct buffer is very expensive; give up.
    return buf;
}
````

1. 首先判断可读字节数，如果是0，之际返回一个空的 Buffer。
2. 获取该 Channel 的 ByteBufAllocator ，如果是直接内存且池化，则分配一个直接内存，将旧的 Buffer 写入到新的中，释放旧的 Buffer。返回新的直接内存 Buffer。
3. 反之，从 FastThreadLocal 中返回一个可重用的直接内存 Buffer，后面和上面的操作一样，写入，删除旧的，返回新的。注意，这里返回的可重用的 Buffer，当调用他的 release 方法的时候，实际上是归还到了 FastThreadLocal 中。对象池的最佳实践。

关于 addMessage 方法，我们将在另一篇文章[Netty 出站缓冲区 ChannelOutboundBuffer 源码解析（isWritable 属性的重要性）](http://thinkinjava.cn/articles/2018/03/18/1521380047606.html)详细阐述，这里只需要知道，他放入了一个 出站缓存中就行了。

## 3. unSafe 的 flush 方法

![](https://upload-images.jianshu.io/upload_images/4236553-15438cc4bf67d74d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

重点是 outboundBuffer.addFlush() 方法和 flush0 方法。

这两个方法在 [Netty 出站缓冲区 ChannelOutboundBuffer 源码解析（isWritable 属性的重要性）](http://thinkinjava.cn/articles/2018/03/18/1521380047606.html)

这里只需要知道，addFlush 方法将刚刚添加进出站 buffer 的数据进行检查，并准备写入 Socket。
flush0 做真正的写入操作，其中，调用了 JDK 的 Socket 的 write 方法，将 ByteBuf 封装的 ByteBuffer 写到 Socket 中。

## 总结

可以看到，数据真正的写出还是调用了 head 的节点中 unsafe 的 write 方法和 flush 方法，其中，write 只是将数据写入到了出站缓冲区，并且，write 方法可以调用多次，flush 才是真正的写入到 Socket。而更详细的细节，可以查看我的另一篇文章[Netty 出站缓冲区 ChannelOutboundBuffer 源码解析（isWritable 属性的重要性）](http://thinkinjava.cn/articles/2018/03/18/1521380047606.html)。

good luck ！！！！























