---
layout: post
title: Netty-心跳服务之-IdleStateHandler-源码分析
date: 2018-03-23 11:11:11.000000000 +09:00
---

![](https://upload-images.jianshu.io/upload_images/4236553-ca22c6142f191e59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言：Netty 提供的心跳介绍

Netty 作为一个网络框架，提供了诸多功能，比如我们之前说的编解码，Netty 准备很多现成的编解码器，同时，Netty 还为我们准备了网络中，非常重要的一个服务-----心跳机制。通过心跳检查对方是否有效，这在 RPC 框架中是必不可少的功能。

Netty 提供了 IdleStateHandler ，ReadTimeoutHandler，WriteTimeoutHandler 检测连接的有效性。当然，你也可以自己写个任务。但我们今天不准备使用自定义任务，而是使用 Netty 内部的。

说以下这三个 handler 的作用。

序 号 |  名称 | 作用
-------|---|---
1    |   IdleStateHandler  | 当连接的空闲时间（读或者写）太长时，将会触发一个 IdleStateEvent 事件。然后，你可以通过你的 ChannelInboundHandler 中重写 userEventTrigged 方法来处理该事件。
2    |  ReadTimeoutHandler | 如果在指定的事件没有发生读事件，就会抛出这个异常，并自动关闭这个连接。你可以在 exceptionCaught 方法中处理这个异常。
3   |  WriteTimeoutHandler  | 当一个写操作不能在一定的时间内完成时，抛出此异常，并关闭连接。你同样可以在 exceptionCaught 方法中处理这个异常。

**注意：**
其中，关于 WriteTimeoutHandler  的描述，著名的 《Netty 实战》和 他的英文原版的描述都过时了，原文描述：
> 如果在指定的时间间隔内没有任何出站数据写入，则抛出一个 WriteTimeoutException.

此书出版的时候，Netty 的文档确实是这样的，但在 2015 年 12 月 28 号的时候，被一个同学修改了逻辑，见下方 git 日志：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-655f41785daa83ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

貌似还是个国人妹子。。。。而现在的文档描述是：

> Raises a {@link WriteTimeoutException} when a write operation cannot finish in a certain period of time.
当一个写操作不能在一定的时间内完成时，就会产生一个 WriteTimeoutException。
 
ReadTimeout 事件和  WriteTimeout 事件都会自动关闭连接，而且，属于异常处理，所以，这里只是介绍以下，我们重点看 IdleStateHandler。

## 1. 什么是 IdleStateHandler

* 回顾一下 IdleStateHandler ：

> 当连接的空闲时间（读或者写）太长时，将会触发一个 IdleStateEvent 事件。然后，你可以通过你的 ChannelInboundHandler 中重写 userEventTrigged 方法来处理该事件。

* 如何使用呢？

> IdleStateHandler 既是出站处理器也是入站处理器，继承了 ChannelDuplexHandler 。通常在 initChannel 方法中将 IdleStateHandler 添加到 pipeline 中。然后在自己的 handler 中重写 userEventTriggered 方法，当发生空闲事件（读或者写），就会触发这个方法，并传入具体事件。
这时，你可以通过 Context 对象尝试向目标 Socekt 写入数据，并设置一个 监听器，如果发送失败就关闭 Socket （Netty  准备了一个 `ChannelFutureListener.CLOSE_ON_FAILURE` 监听器用来实现关闭 Socket 逻辑）。
这样，就实现了一个简单的心跳服务。



*****



## 2. 源码分析

* ##### 1.构造方法，该类有 3 个构造方法，主要对一下 4 个属性赋值：

````java
private final boolean observeOutput;// 是否考虑出站时较慢的情况。默认值是false（不考虑）。
private final long readerIdleTimeNanos; // 读事件空闲时间，0 则禁用事件
private final long writerIdleTimeNanos;// 写事件空闲时间，0 则禁用事件
private final long allIdleTimeNanos; //读或写空闲时间，0 则禁用事件
````

* #####  2. handlerAdded 方法

当该 handler 被添加到 pipeline 中时，则调用 initialize 方法：

````java
private void initialize(ChannelHandlerContext ctx) {
    switch (state) {
    case 1:
    case 2:
        return;
    }
    state = 1;
    initOutputChanged(ctx);

    lastReadTime = lastWriteTime = ticksInNanos();
    if (readerIdleTimeNanos > 0) {
      // 这里的 schedule 方法会调用 eventLoop 的 schedule 方法，将定时任务添加进队列中
        readerIdleTimeout = schedule(ctx, new ReaderIdleTimeoutTask(ctx),
                readerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (writerIdleTimeNanos > 0) {
        writerIdleTimeout = schedule(ctx, new WriterIdleTimeoutTask(ctx),
                writerIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
    if (allIdleTimeNanos > 0) {
        allIdleTimeout = schedule(ctx, new AllIdleTimeoutTask(ctx),
                allIdleTimeNanos, TimeUnit.NANOSECONDS);
    }
}
````

只要给定的参数大于0，就创建一个定时任务，每个事件都创建。同时，将 state 状态设置为 1，防止重复初始化。调用 initOutputChanged 方法，初始化 “监控出站数据属性”，代码如下：

````java
private void initOutputChanged(ChannelHandlerContext ctx) {
    if (observeOutput) {
        Channel channel = ctx.channel();
        Unsafe unsafe = channel.unsafe();
        ChannelOutboundBuffer buf = unsafe.outboundBuffer();
        // 记录了出站缓冲区相关的数据，buf 对象的 hash 码，和 buf 的剩余缓冲字节数
        if (buf != null) {
            lastMessageHashCode = System.identityHashCode(buf.current());
            lastPendingWriteBytes = buf.totalPendingWriteBytes();
        }
    }
}
````
首先说说这个 observeOutput  “监控出站数据属性” 的作用。因为 github 上有人提了 issue ，[issue 地址](https://github.com/netty/netty/issues/6150)，本来是没有这个参数的。为什么需要呢？

假设：当你的客户端应用每次接收数据是30秒，而你的写空闲时间是 25 秒，那么，当你数据还没有写出的时候，写空闲时间触发了。实际上是不合乎逻辑的。因为你的应用根本不空闲。

怎么解决呢？

Netty 的解决方案是：记录最后一次输出消息的相关信息，并使用一个值 firstXXXXIdleEvent 表示是否再次活动过，每次读写活动都会将对应的 first 值更新为 true，如果是 false，说明这段时间没有发生过读写事件。同时如果第一次记录出站的相关数据和第二次得到的出站相关数据不同，则说明数据在缓慢的出站，就不用触发空闲事件。

总的来说，这个字段就是用来对付 “客户端接收数据奇慢无比，慢到比空闲时间还多” 的极端情况。所以，Netty 默认是关闭这个字段的。



* ##### 3. 该类内部的 3 个定时任务类

如下图：

![](https://upload-images.jianshu.io/upload_images/4236553-c64efc5485213104.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这 3 个定时任务分别对应 读，写，读或者写 事件。共有一个父类。这个父类提供了一个模板方法：


![](https://upload-images.jianshu.io/upload_images/4236553-5b534402509a5ba6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当通道关闭了，就不执行任务了。反之，执行子类的 run 方法。

**1. 读事件的 run 方法**

代码如下：

````java
protected void run(ChannelHandlerContext ctx) {
    long nextDelay = readerIdleTimeNanos;
    if (!reading) {
        nextDelay -= ticksInNanos() - lastReadTime;
    }

    if (nextDelay <= 0) {
        // Reader is idle - set a new timeout and notify the callback.
        // 用于取消任务 promise
        readerIdleTimeout = schedule(ctx, this, readerIdleTimeNanos, TimeUnit.NANOSECONDS);

        boolean first = firstReaderIdleEvent;
        firstReaderIdleEvent = false;

        try {
            // 再次提交任务
            IdleStateEvent event = newIdleStateEvent(IdleState.READER_IDLE, first);
            // 触发用户 handler use
            channelIdle(ctx, event);
        } catch (Throwable t) {
            ctx.fireExceptionCaught(t);
        }
    } else {
        // Read occurred before the timeout - set a new timeout with shorter delay.
        readerIdleTimeout = schedule(ctx, this, nextDelay, TimeUnit.NANOSECONDS);
    }
}
````

该方法很简单：
1. 得到用户设置的超时时间。
2. 如果读取操作结束了（执行了 channelReadComplete 方法设置） ，就用当前时间减去给定时间和最后一次读操作的时间（执行了 channelReadComplete 方法设置），如果小于0，就触发事件。反之，继续放入队列。间隔时间是新的计算时间。
3. 触发的逻辑是：首先将任务再次放到队列，时间是刚开始设置的时间，返回一个 promise 对象，用于做取消操作。然后，设置 first 属性为 false ，表示，下一次读取不再是第一次了，这个属性在 channelRead 方法会被改成 true。
4. 创建一个 IdleStateEvent  类型的写事件对象，将此对象传递给用户的 UserEventTriggered 方法。完成触发事件的操作。

总的来说，每次读取操作都会记录一个时间，定时任务时间到了，会计算当前时间和最后一次读的时间的间隔，如果间隔超过了设置的时间，就触发 UserEventTriggered 方法。就是这么简单。


再看看写事件任务。

**2. 写事件的 run 方法**

写任务的逻辑基本和读任务的逻辑一样，唯一不同的就是有一个针对 出站较慢数据的判断。

````java
 if (hasOutputChanged(ctx, first)) {
     return;
}
````

如果这个方法返回 true，就不执行触发事件操作了，即使时间到了。看看该方法实现：

````java
private boolean hasOutputChanged(ChannelHandlerContext ctx, boolean first) {
    if (observeOutput) {
        // 如果最后一次写的时间和上一次记录的时间不一样，说明写操作进行过了，则更新此值
        if (lastChangeCheckTimeStamp != lastWriteTime) {
            lastChangeCheckTimeStamp = lastWriteTime;
            // 但如果，在这个方法的调用间隙修改的，就仍然不触发事件
            if (!first) { // #firstWriterIdleEvent or #firstAllIdleEvent
                return true;
            }
        }
        Channel channel = ctx.channel();
        Unsafe unsafe = channel.unsafe();
        ChannelOutboundBuffer buf = unsafe.outboundBuffer();
        // 如果出站区有数据
        if (buf != null) {
            // 拿到出站缓冲区的 对象 hashcode
            int messageHashCode = System.identityHashCode(buf.current());
            // 拿到这个 缓冲区的 所有字节
            long pendingWriteBytes = buf.totalPendingWriteBytes();
            // 如果和之前的不相等，或者字节数不同，说明，输出有变化，将 "最后一个缓冲区引用" 和 “剩余字节数” 刷新
            if (messageHashCode != lastMessageHashCode || pendingWriteBytes != lastPendingWriteBytes) {
                lastMessageHashCode = messageHashCode;
                lastPendingWriteBytes = pendingWriteBytes;
                // 如果写操作没有进行过，则任务写的慢，不触发空闲事件
                if (!first) {
                    return true;
                }
            }
        }
    }
    return false;
}
````

写了一些注释，还是再梳理一下吧：
1. 如果用户没有设置了需要观察出站情况。就返回 false，继续执行事件。
2. 反之，继续向下， 如果最后一次写的时间和上一次记录的时间不一样，说明写操作刚刚做过了，则更新此值，但仍然需要判断这个 first 的值，如果这个值还是 false，说明在这个写事件是在两个方法调用间隙完成的 / 或者是第一次访问这个方法，就仍然不触发事件。
3. 如果不满足上面的条件，就取出缓冲区对象，如果缓冲区没对象了，说明没有发生写的很慢的事件，就触发空闲事件。反之，记录当前缓冲区对象的 hashcode 和 剩余字节数，再和之前的比较，如果任意一个不相等，说明数据在变化，或者说数据在慢慢的写出去。那么就更新这两个值，留在下一次判断。
4. 继续判断 first ，如果是 fasle，说明这是第二次调用，就不用触发空闲事件了。


整个逻辑如下：

![流程图](https://upload-images.jianshu.io/upload_images/4236553-131573522cdb3942.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这里有个问题，为什么第一次的时候一定要触发事件呢？假设，客户端开始变得很慢，这个时候，定时任务监听发现时间到了，就进入这里判断，当上次记录的缓冲区相关数据已经不同，这个时候难道触发事件吗？

实际上，这里是 Netty 的一个考虑：假设真的发生了很写出速度很慢的问题，很可能引发 OOM，相比叫连接空闲，这要严重多了。为什么第一次一定要触发事件呢？如果不触发，用户根本不知道发送了什么，当一次写空闲事件触发，随后出现了 OOM，用户可以感知到：可能是写的太慢，后面的数据根本写不进去，所以发生了OOM。所以，这里的一次警告还是必要的。

当然，这是我的一个猜测。有必要的话，可以去 Netty 那里提个 issue。

好，关于客户端写的慢的特殊处理告一段落。再看看另一个任务的逻辑。

**3. 所有事件的 run 方法**

这个类叫做 AllIdleTimeoutTask ，表示这个监控着所有的事件。当读写事件发生时，都会记录。代码逻辑和写事件的的基本一致，除了这里：

````java
long nextDelay = allIdleTimeNanos;
if (!reading) {
   // 当前时间减去 最后一次写或读 的时间 ，若大于0，说明超时了
   nextDelay -= ticksInNanos() - Math.max(lastReadTime, lastWriteTime);
}
````

这里的时间计算是取读写事件中的最大值来的。然后像写事件一样，判断是否发生了写的慢的情况。最后调用 ctx.fireUserEventTriggered(evt) 方法。


通常这个使用的是最多的。构造方法一般是：

````java
pipeline.addLast(new IdleStateHandler(0, 0, 30, TimeUnit.SECONDS));
````
读写都是 0 表示禁用，30 表示 30 秒内没有任务读写事件发生，就触发事件。注意，当不是 0 的时候，这三个任务会重叠。


## 总结

IdleStateHandler 可以实现心跳功能，当服务器和客户端没有任何读写交互时，并超过了给定的时间，则会触发用户 handler 的 userEventTriggered 方法。用户可以在这个方法中尝试向对方发送信息，如果发送失败，则关闭连接。

IdleStateHandler  的实现基于 EventLoop 的定时任务，每次读写都会记录一个值，在定时任务运行的时候，通过计算当前时间和设置时间和上次事件发生时间的结果，来判断是否空闲。

内部有 3 个定时任务，分别对应读事件，写事件，读写事件。通常用户监听读写事件就足够了。

同时，IdleStateHandler  内部也考虑了一些极端情况：`客户端接收缓慢，一次接收数据的速度超过了设置的空闲时间`。Netty 通过构造方法中的 observeOutput 属性来决定是否对出站缓冲区的情况进行判断。

如果出站缓慢，Netty 不认为这是空闲，也就不触发空闲事件。但第一次无论如何也是要触发的。因为第一次无法判断是出站缓慢还是空闲。当然，出站缓慢的话，OOM 比空闲的问题更大。

所以，当你的应用出现了内存溢出，OOM之类，并且写空闲极少发生（使用了 observeOutput 为 true），那么就需要注意是不是数据出站速度过慢。

默认 observeOutput  是 false，意思是，即使你的应用出站缓慢，Netty 认为是写空闲。

可见这个 observeOutput 的作用好像不是那么重要，如果真的发生了出站缓慢，判断是否空闲根本就不重要了，重要的是 OOM。所以 Netty 选择了默认 false。

还有一个注意的地方：刚开始我们说的 ReadTimeoutHandler ，就是继承自 IdleStateHandler，当触发读空闲事件的时候，就触发 ctx.fireExceptionCaught 方法，并传入一个 ReadTimeoutException，然后关闭 Socket。

而 WriteTimeoutHandler 的实现不是基于 IdleStateHandler 的，他的原理是，当调用 write 方法的时候，会创建一个定时任务，任务内容是根据传入的 promise 的完成情况来判断是否超出了写的时间。当定时任务根据指定时间开始运行，发现 promise 的 isDone 方法返回 false，表明还没有写完，说明超时了，则抛出异常。当 write 方法完成后，会打断定时任务。

好了，关于 Netty 自带的心跳相关的类就介绍到这里。这些功能对于开发稳定的高性能 RPC 至关重要。

good luck！！！


