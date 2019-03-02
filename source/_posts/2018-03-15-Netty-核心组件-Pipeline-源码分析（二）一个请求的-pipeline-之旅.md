---
layout: post
title: Netty-核心组件-Pipeline-源码分析（二）一个请求的-pipeline-之旅
date: 2018-03-15 11:11:11.000000000 +09:00
---

![](https://upload-images.jianshu.io/upload_images/4236553-a4c1578193ba9b4d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


目录大纲：
1. 前言
2. 针对 Netty 例子源码做了哪些修改？
3. 看 pipeline 是如何将数据送到自定义 handler 的
4. 看 pipeline 是如何将数据从自定义 handler 送出的
5. 总结


## 前言
在 [Netty 核心组件 Pipeline 源码分析（一）之剖析 pipeline 三巨头](https://www.jianshu.com/p/18e53b6b224f) 中，我们详细阐述了 pipeline，context，handler 的设计与实现。知道了 Netty 是如何处理网络数据的，但到目前为止，我们都没有实打实的走一遍流程，实际上，debug 一遍流程，会让我们对 Netty 处理整个数据流更加深刻理解。


楼主此次使用的依然还是 Netty 自带的 ServerExample 和 Client Example，我想大家应该早就下好源码了吧。当然，针对源码，我们也做了一些修改，方便让我们更加的容易测试。




## 1. 针对 Netty 例子源码做了哪些修改？

针对 EchoInServerHandler 的channelRead 方法做了如下修改：

![](https://upload-images.jianshu.io/upload_images/4236553-fdc1b8d5389d5733.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

读取客户端发送来的数据，并打印，然后发送一串字符串给客户端。当然，其余方法都加入了日志打印。

针对 EchoClientHandler  的 channelActive 方法做了如下修改：
![](https://upload-images.jianshu.io/upload_images/4236553-29742911b277d575.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当连接服务器成功时，发送一串字符串。

针对 EchoClientHandler 的 channelRead 方法做了如下修改：
![](https://upload-images.jianshu.io/upload_images/4236553-1e6ca8bca7225eca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解码客户端发送来的数据并打印。

同时新增了一个 EchoOutServerHandler 类，继承了 ChannelOutboundHandlerAdapter 类，用于打印出站事件：
![](https://upload-images.jianshu.io/upload_images/4236553-b75b1bbe7b32a870.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行后的结果如下：

Server 控制台：
![](https://upload-images.jianshu.io/upload_images/4236553-00bf8f5a7be44ab4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Client 控制台：
![](https://upload-images.jianshu.io/upload_images/4236553-d33d691218b7daa7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上面红色字可以看出，打印出了我们想要的结果，Server 接收到了 Client 的信息并打印，Client 接收到了 Server 的信息并打印。


下面就让我们 debug，看看一个请求是如何在 pipeline 中游走的吧！


## 2. 看 pipeline 是如何将数据送到自定义 handler 的

首先我们 debug 模式启动 EchoServer，让整个 Server 处于待命状态。断点打在 EventLoop 类的 processSelectedKey 方法中，监听 accpet 事件和 read 事件。

![](https://upload-images.jianshu.io/upload_images/4236553-99ed3ae33b56b29a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时启动客户端，这个时候 Server 断点开始卡住，我们开始 debug。

![](https://upload-images.jianshu.io/upload_images/4236553-12b446587ef0d477.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里的 readOps 是16，Accept 事件，这里的 unsafe 是 ServerSocket 的 unsafe，如果还记的 [Netty 接受请求过程源码分析 (基于4.1.23)](https://www.jianshu.com/p/c83de9575825) 文中所说，在这之后，会创建一个 客户端的 ChannelSocket，然后该 Socket 会向 selector 注册读事件，所以，我们这里需要放开断点，得到读事件才是真正请求的开始。

好，我们使用 IDEA 的 Force run to cursor 功能，让线程直接卡到这里，这时，你会发现，EventLoop-3-1 卡住了，而不是之前的 EventLoop-2-1，3-1 是上面线程大家应该知道吧，就是 worker group 线程池中的 eventLoop，也就是刚刚注册的 Socket。

![](https://upload-images.jianshu.io/upload_images/4236553-756d646f6b69e535.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面的断点可以看出，这里确实是读事件，断点提示也指出这个 unsafe 是 NioSocketChannel 的 内部类 NioSocketChannelUnsafe，我们跟进去看看。

进入的是 NioSocketChannelUnsafe 的抽象父类 AbstractNioByteChannel 的 read 方法。精简过的代码如下：

````java
public final void read() {
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator();
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();

    // 读取数据到容器
    byteBuf = allocHandle.allocate(allocator);
    allocHandle.lastBytesRead(doReadBytes(byteBuf));
    // 让 handler 处理容器中的数据
    pipeline.fireChannelRead(byteBuf);

    // 告诉容器处理完毕了，触发完成事件
    pipeline.fireChannelReadComplete();

}
````

这里楼主简化了很多代码，留下的是对本次分析比较重要的内容。注释已经写的很清除，首先从 unsafe 中读取数据，然后，将读好的数据交给 pipeline，pipeline 调用 inbound 的 channelRead 方法，读取成功后，调用 inbound 的 handler 的 ChannelReadComplete 方法。

在进入方法之前，楼主向祭出上文中的图，让我们看后面的代码更清晰：

![](https://upload-images.jianshu.io/upload_images/4236553-6955360ea06b490e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该图诠释了一个请求在 pipeline 的流动过程。请记住他。



整个过程还是比较清晰的。我们首先进入 pipeline 的 fireChannelReadComplete 方法，这个方法是实现了 invoker 的方法。

![](https://upload-images.jianshu.io/upload_images/4236553-f84b26476da61a18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

内部调用的是 AbstractChannelHandlerContext.invokeChannelRead(head, msg) 静态方法，并传入了 head，我们知道入站数据都是从 head 开始的，以保证后面所有的 handler 都由机会处理数据流。

我们看看这个静态方法内部是怎么样的：


````java
    static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRead(m);
        } else {
            executor.execute(new Runnable() {
                public void run() {
                    next.invokeChannelRead(m);
                }
            });
        }
    }
````

调用这个 Context （也就是 head） 的 invokeChannelRead 方法，并传入数据。我们再看看 invokeChannelRead 方法的实现：


````java
    private void invokeChannelRead(Object msg) {
        if (invokeHandler()) {
            try {
                ((ChannelInboundHandler) handler()).channelRead(this, msg);
            } catch (Throwable t) {
                notifyHandlerException(t);
            }
        } else {
            fireChannelRead(msg);
        }
    }
````

这里和我们的图画的是一致的，调用了 Context 包装的 handler 的 channelRead 方法。注意：直到目前，这个  Context 还是 head，也就是调用 head 的 channelRead 方法。那么这个方法是怎么实现的呢？

````java
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            ctx.fireChannelRead(msg);
        }
````

什么都没做，和我们图中一样，调用 Context 的 fire 系列方法，将请求转发给下一个节点。我们这里是 fireChannelRead 方法，注意，这里方法名字都挺像的。需要细心区分。下面我们看看 Context 的成员方法 fireChannelRead：

````java
    @Override
    public ChannelHandlerContext fireChannelRead(final Object msg) {
        invokeChannelRead(findContextInbound(), msg);
        return this;
    }
````

这个是 head 的抽象父类 AbstractChannelHandlerContext 的实现，该方法再次调用了静态 fire 系列方法，但和上次不同的是，不再放入 head 参数了，而是使用 findContextInbound 方法的返回值。从这个方法的名字可以看出，是找到入站类型的 handler。我们看看方法实现：

````java
    private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;
        } while (!ctx.inbound);
        return ctx;
    }
````

该方法很简单，找到当前 Context 的 next 节点（inbound 类型的）并返回。这样就能将请求传递给后面的 inbound handler 了。

重复上面的逻辑，终于数据到了我们自己写的 handler-------EchoInServerHandler。

![](https://upload-images.jianshu.io/upload_images/4236553-eede9c0daf8ed698.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


好，到这里，我们已经知道了一个请求时怎么到达我们自定义的 handler 的，再来看看我们的图：

![](https://upload-images.jianshu.io/upload_images/4236553-30b277ac753eab11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


请求进来时，pipeline 会从 head 节点开始输送，通过配合 invoker 接口的 fire 系列方法，实现 Context 链在 pipeline 中的完美传递。最终到达我们自定义的 handler。 


到了自定义 handler，我们会输出客户端发送的内容，我们截图看看：

![](https://upload-images.jianshu.io/upload_images/4236553-5b08aaba119302cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


成功输出。

注意：此时如果我们想继续向后传递该怎么办呢？我们前面说过，可以调用 Context 的 fire 系列方法，就像 head 的 channelRead 方法一样，调用 fire 系列方法，直接向后传递就 ok 了。

当然，我们这里不需要，我们需要发送一条数据客户端。那么，我们就来看看一条数据是如何到达客户端的。






## 3. 看 pipeline 是如何将数据从自定义  handler 送出的

在打印了客户端的内容后，我们调用了 Context 的 writeAndFlush 方法，从 inbound 和 outbound 的定义来看，这个方法是 outbound 定义的，也就是出站方法。

在debug 进去看看之前，我们能否猜测一下呢，这个 Context 肯定会调用他的抽象父类  AbstractChannelHandlerContext 方法， 我们跟进去看看：

![](https://upload-images.jianshu.io/upload_images/4236553-5afd30f50fc5c7d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

果不其然。调用了 AbstractChannelHandlerContext  的 writeAndFlush 方法，然后，调用了他的重载方法，多传入了一个 promise 实例。看看是如何创建的：

````java
  @Override
    public ChannelPromise newPromise() {
        return new DefaultChannelPromise(channel(), executor());
    }
````

我们再跟进去看看 writeAndFlush ：

![](https://upload-images.jianshu.io/upload_images/4236553-85e7a0488be896be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里调用了 write 方法，并直接返回了 promise。继续跟进查看：

![](https://upload-images.jianshu.io/upload_images/4236553-716b4681c5144259.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


注意：这里调用了 findContextOutbound，寻找下一个 outbound 节点。我们看看是如何实现的：

![](https://upload-images.jianshu.io/upload_images/4236553-9249357897f47f61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据当前节点，找到之前的节点并且是 outbound 类型。

可以看到，数据开始出站，从后向前开始流动，和入站的方向是反的。

回到 write 方法，得到下一个节点后，调用下一个节点的 invokeWriteAndFlush 方法，这个是 invoker 接口的方法。

![](https://upload-images.jianshu.io/upload_images/4236553-8bd120ca2114c6ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

调用 invokeWrite0 方法，注意，Netty 很多方法都以 0 结尾，表示这是最底层的方法了，而再 JDK 中，结尾是 0 表示这是一个本地方法。我们进入该方法查看：

![](https://upload-images.jianshu.io/upload_images/4236553-9296cc974cfb2626.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

调用了这个 Context 的 worite 方法。还记得我们也写了一个 EchoOutServerHandler 类吗，可能会进入我们自己写入的类的方法吗？当然不会，因为我们添加的顺序是下面这样的：

![](https://upload-images.jianshu.io/upload_images/4236553-c592f9ffa68759c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

inbound 在前，outbound 在后，当程序走到 inbound 就调用 outbound 的方法了，并找当前节点的上一个节点，而我们写的 outbound 是这个节点的下一个节点，永远不会走到这里的。

那么会走到哪里呢，当然是走到 head 节点，因为 head 节点就是 outbound 类型的 handler。

进入到 head 的 write 方法查看：

![](https://upload-images.jianshu.io/upload_images/4236553-f46eaec28fb8a213.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

调用了 底层的 unsafe 操作数据，到这里，我们就不跟了，基于我们今天的目的，我们只想知道一个请求在 pipeline 是如何流转的。底层数据传播的细节就不再赘述。留在以后研究。

当执行完这个 write 方法后，方法开始退栈。逐步退到 unsafe 的 read 方法，回到最初开始的地方，然后继续调用  pipeline.fireChannelReadComplete() 方法，重复之前 pipeline 的设计。


![](https://upload-images.jianshu.io/upload_images/4236553-931b09e2ca0d0a0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

到这里，我们应该已经清楚了一个请求时如何在 pipeline 中周转的了。

## 4. 总结

总结一下一个请求在 pipeline 中的流转过程：
1. 调用 pipeline 的 fire 系列方法，这些方法是接口 invoker 设计的，pipeline 实现了 invoker 的所有方法，inbound 事件从 head 开始流入，outbound 事件从 tail 开始流出。
2. pipeline 会将请求交给 Context，然后 Context 通过抽象父类 AbstractChannelHandlerContext 的 invoke 系列方法（静态和非静态的）配合 AbstractChannelHandlerContext 的 fire 系列方法再配合 findContextInbound 和 findContextOutbound 方法完成各个 Context 的数据流转。
3. 当入站过程中，调用 了出站的方法，那么请求就不会向后走了。后面的处理器将不会有任何作用。想继续相会传递就调用 Context 的 fire 系列方法，让 Netty 在内部帮你传递数据到下一个节点。如果你想在整个通道传递，就在 handler 中调用 channel 或者 pipeline 的对应方法，这两个方法会将数据从头到尾或者从尾到头的流转一遍。

最后，再次祭上我们的图，配合 debug 堆栈信息：


![](https://upload-images.jianshu.io/upload_images/4236553-59d16c1e00f7847b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上图就是 pipeline 一个通用的数据流动过程。

好。good luck ！！！！
