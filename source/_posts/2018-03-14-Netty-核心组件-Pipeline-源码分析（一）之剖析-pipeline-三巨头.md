---
layout: post
title: Netty-核心组件-Pipeline-源码分析（一）之剖析-pipeline-三巨头
date: 2018-03-14 11:11:11.000000000 +09:00
---
![](https://upload-images.jianshu.io/upload_images/4236553-46e75c0dee796ef1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

目录大纲：
0. 前言
1. ChannelPipeline | ChannelHandler | ChannelHandlerContext 三巨头介绍
2. 三巨头编织过程（创建过程）
3. ChannelPipeline 是如何调度 handler 的
4. 总结

## 前言


相信对 Netty 熟悉的同学对 pipeline 都非常的熟悉，肯定也有不熟悉的，不管怎样，楼主今天的目的就是将 pipeline 从头撸到尾，彻彻底底的理解 pipeline 的每一步操作。

当然，文章还是一如既往的长。请非战斗人员尽快撤离！！！！

让我们开始吧！

## 1.  ChannelPipeline | ChannelHandler | ChannelHandlerContext 三巨头介绍

如果把 Netty 比作一个人类的话，那么 EventLoop 就是这个人的大脑，负责这个人的所有操作。而 pipeline 就是这个的肠道，负责将这个人吃进去的东西进行消化然后处理。这个比喻可能不是很恰当，当然这也是为了加深理解。

当然，我说的 pipelie 是一个广义的概念，pipeline 包括很多东西，就像我们标题说的三巨头，下面我们就来好好说说他们的关系。

#### 1.0三者关系

我们在之前的文章中知道，每当 ServerSocket 创建一个新的连接，就会创建一个 Socket，对应的就是目标客户端。而每一个新创建的 Socket 都将会分配一个全新的 ChannelPipeline（以下简称 pipeline），他们的关系是永久不变的；而每一个 ChannelPipeline 内部都含有多个 ChannelHandlerContext（以下简称 Context），他们一起组成了双向链表，这些 Context 用于包装我们调用 addLast 方法时添加的 ChannelHandler（以下简称 handler）。

所以说，他们的关系是这样的：

![](https://upload-images.jianshu.io/upload_images/4236553-3c1e4da3ceaf3559.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中：ChannelSocket 和 ChannelPipeline 是一对一的关联关系，而 pipeline 内部的多个 Context 形成了链表，Context 只是对 Handler 的封装。

为什么需要对 Handler 进行封装呢？想象一下：当你  A handler 要调 B handler 方法的时候，如果没有 Context，那么就直接调用了，如果有一些需要在调用前后通用的逻辑就需要在每个 handler 地方都写，这样会导致代码重复，而且紧耦合，不符合设计原则。

总的来说，当一个请求进来的时候，会进入 Socket 对应的 pipeline，并流经 pipeline 所有的 handler，对，就是设计模式中的过滤器模式，可以说是最佳实践。用过滤器处理网络数据的不止 netty，还有 tomcat，相信大家对 tomcat 的 filter（应该是 servlet 的 filter） 都非常的熟悉吧。

知道了他们的概念，我们继续深入看看他们的设计。

#### 1.1 ChannelPipeline 作用及设计

首先看 pipeline 的接口设计：

![](https://upload-images.jianshu.io/upload_images/4236553-c179e6a51e6d0896.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


````java
public interface ChannelPipeline
    extends ChannelInboundInvoker, ChannelOutboundInvoker, Iterable<Entry<String, ChannelHandler>> {

  ChannelPipeline addFirst(String name, ChannelHandler handler);
  ChannelPipeline addAfter(String baseName, String name, ChannelHandler handler);
  ChannelPipeline addBefore(String baseName, String name, ChannelHandler handler);
  ChannelPipeline addLast(ChannelHandler... handlers);
  Channel channel();
  ChannelHandlerContext context(ChannelHandler handler);
  ChannelPipeline remove(ChannelHandler handler);
  ChannelPipeline replace(ChannelHandler oldHandler, String newName, ChannelHandler newHandler);
}
````

通过 UML 图，可以看到该接口继承了 inBound，outBound，Iterable 接口，表示他可以调用当数据出站的方法和入站的方法，同时也能遍历内部的链表。

再看看他的几个具有代表性的方法，基本上都是针对 handler 链表的插入，追加，删除，替换操作，甚至，我们可以想象他就是一个 LinkedList。同时，他也能返回 channel（也就是 socket）。

在 pipeline 的接口文档上，作者写了很多注释并且画了一幅图：

![](https://upload-images.jianshu.io/upload_images/4236553-180e212b5357414f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**文档大致意思是：**

 这是一个 handler 的 list，handler 用于处理或拦截入站事件和出站事件，pipeline 实现了过滤器的高级形式，以便用户完全控制事件如何处理以及 handler 在 pipeline 中如何交互。

上图描述了一个典型的  handler 在 pipeline 中处理 I/O 事件的方式，IO 事件由 inboundHandler 或者 outBoundHandler 处理，并通过调用 ChannelHandlerContext.fireChannelRead 方法转发给其最近的处理程序 。

入站事件由入站处理程序以自下而上的方向处理，如图所示。入站处理程序通常处理由图底部的I / O线程生成入站数据。入站数据通常从如 SocketChannel.read(ByteBuffer) 获取。如果入站事件超出顶层入站处理程序，它将被静默放弃，或者在需要您关注时进行记录。

通常一个 pipeline 有多个 handler，例如，一个典型的服务器在每个通道的管道中都会有以下处理程序，但是您的里程可能会因协议和业务逻辑的复杂性和特征而异：

1.  协议解码器 - 将二进制数据（例如 ByteBuf 在io.netty.buffer中的类)）转换为Java对象。
2.  协议编码器 - 将Java对象转换为二进制数据。
3.  业务逻辑处理程序 - 执行实际业务逻辑（例如数据库访问）。

注意：你的业务程序不能将线程阻塞，他将会影响 IO 的速度，进而影响整个 Netty 程序的性能。如果你的业务程序很快，就可以放在 IO 线程中，反之，你需要异步执行。或者在添加 handler 的时候添加一个线程池，例如：

````java
// 下面这个任务执行的时候，将不会阻塞 IO 线程，执行的线程来自 group 线程池
 pipeline.addLast（group，“handler”，new MyBusinessLogicHandler（））;
````

好，关于 pipeline 的设计就介绍到这里。我们再看看我们常见的 ChannelHandler。



#### 1.2 ChannelHandler  作用及设计

关于 ChannelHanderl 我们都非常的熟悉吧，在每个最初认识 Netty 的人都知道他的 demo 程序中会添加 handler 并自己实现 handler，通常，我们说 handler 指的就是 ChannelHandler。

ChannelHandler 是一个顶级接口，没有继承任何接口：
![](https://upload-images.jianshu.io/upload_images/4236553-4efe995fcd157b4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

定义了 3 个方法：

````java
public interface ChannelHandler {
    // 当把 ChannelHandler 添加到 pipeline 时被调用
    void handlerAdded(ChannelHandlerContext ctx) throws Exception;
    // 当从 pipeline 中移除时调用
    void handlerRemoved(ChannelHandlerContext ctx) throws Exception;
    // 当处理过程中在 pipeline 发生异常时调用
    @Deprecated
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
}
````

总的来说，ChannelHandler 的作用就是处理 IO 事件或拦截 IO 事件，并将其转发给下一个处理程序 ChannelHandler。

从上面的代码中，可以看到，ChannelHandler 并没有提供很多的方法，因为 Handler 处理事件时分入站和出站的，两个方向的操作都是不同的，因此，Netty 定义了两个子接口继承 ChannelHandler。

**1. ChannelInboundHandler 入站事件接口**

````java
public interface ChannelInboundHandler extends ChannelHandler {

    void channelRegistered(ChannelHandlerContext ctx) throws Exception;
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;
    void channelActive(ChannelHandlerContext ctx) throws Exception;
    void channelInactive(ChannelHandlerContext ctx) throws Exception;
    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;
    void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;
    void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
}
````


如果你经常使用 Netty 程序，你会非常的熟悉这些方法，比如 channelActive 用于当 Channel 处于活动状态时被调用；channelRead ------ 当从Channel 读取数据时被调用等等方法。通常我们需要重写一些方法，当发生关注的事件，我们需要在方法中实现我们的业务逻辑，因为当事件发生时，Netty 会回调对应的方法。

注意：当你重写了上面的 channelRead 方法时，你需要显示的释放与池化的 ByteBuf 实例相关的内存。Netty 为此提供了了一个使用方法 ReferenceCountUtil.release().


**2. ChannelOutboundHandler 出站事件接口**

ChannelOutboundHandler  负责出站操作和处理出站数据。接口方法如下：

````java
public interface ChannelOutboundHandler extends ChannelHandler {

    void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception;
    void connect(
            ChannelHandlerContext ctx, SocketAddress remoteAddress,
            SocketAddress localAddress, ChannelPromise promise) throws Exception;
    void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void read(ChannelHandlerContext ctx) throws Exception;
    void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;
    void flush(ChannelHandlerContext ctx) throws Exception;
}
````

大家可以熟悉熟悉这个接口，比如 bind 方法，当请求将 Channel 绑定到本地地址时调用，close 方法，当请求关闭 Channel 时调用等等，总的来说，出站操作都是一些连接和写出数据类似的方法。和入站操作有很大的不同。

总之，我们要区别入站方法和出站方法，这在 pipeline 中将会起很大的作用。

**3. ChannelDuplexHandler 处理出站和入站事件**

````java
public class ChannelDuplexHandler extends ChannelInboundHandlerAdapter implements ChannelOutboundHandler {

    public void bind(ChannelHandlerContext ctx, SocketAddress localAddress,
                     ChannelPromise promise) throws Exception {
        ctx.bind(localAddress, promise);
    }
    public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress,
                        SocketAddress localAddress, ChannelPromise promise) throws Exception {
        ctx.connect(remoteAddress, localAddress, promise);
    }
    public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise)
            throws Exception {
        ctx.disconnect(promise);
    }
    public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
        ctx.close(promise);
    }
    public void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
        ctx.deregister(promise);
    }
    public void read(ChannelHandlerContext ctx) throws Exception {
        ctx.read();
    }
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        ctx.write(msg, promise);
    }
    public void flush(ChannelHandlerContext ctx) throws Exception {
        ctx.flush();
    }
}

````
从上面的代码中可以看出 ChannelDuplexHandler 间接实现了入站接口并直接实现了出站接口。是一个通用的能够同时处理入站事件和出站事件的类。


介绍了完了  ChannelHandler 的设计，我们再来看看 ChannelHandlerContext 。

#### 1.3 ChannelHandlerContext 作用及设计

实际上，从上面的代码中，我们已经看到了 Context 的用处，在 ChannelDuplexHandler 中，cxt 无处不在。事实上，以read 方法为例：调用 handler 的 read 方法，如果你不处理，就会调用 context 的 read 方法，context 再调用下一个 context 的 handler 的 read 方法。

我们看看 ChannelHandlerContext 的接口 UML :

![](https://upload-images.jianshu.io/upload_images/4236553-8adb03da878b19de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ChannelHandlerContext  继承了出站方法调用接口和入站方法调用接口。那么， ChannelInboundInvoker 和 ChannelOutboundInvoker 又有哪些方法呢？

![ChannelInboundInvoker 入站方法调用器](https://upload-images.jianshu.io/upload_images/4236553-e3c5145d0a288528.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![ChannelOutboundInvoker 出站方法调用器](https://upload-images.jianshu.io/upload_images/4236553-94cd030d6ee81d4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，这两个 invoker 就是针对入站或出站方法来的，就是再 入站或出站 handler 的外层再包装一层，达到在方法前后拦截并做一些特定操作的目的。

而 ChannelHandlerContext 不仅仅时继承了他们两个的方法，同时也定义了一些自己的方法：

````java
public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker {

    Channel channel();
    EventExecutor executor();
    String name();
    ChannelHandler handler();
    boolean isRemoved();
    ChannelPipeline pipeline();
    ByteBufAllocator alloc();
}
````

这些方法能够获取 Context 上下文环境中对应的比如 channel，executor，handler ，pipeline，内存分配器，关联的 handler 是否被删除。


我们可以认为，Context 就是包装了 handler 相关的一切，以方便 Context 可以在 pipeline 方便的操作 handler 相关的资源和行为。


## 2. 三巨头编织过程（创建过程）

介绍完了 "三巨头" 的接口设计和一些方法，那么我们就看看，他们是如何编制在一起的。

在文章前面，我们说：

>  每当 ServerSocket 创建一个新的连接，就会创建一个 Socket，对应的就是目标客户端。而每一个新创建的 Socket 都将会分配一个全新的 ChannelPipeline（以下简称 pipeline），他们的关系是永久不变的；而每一个 ChannelPipeline 内部都含有多个 ChannelHandlerContext（以下简称 Context），他们一起组成了双向链表，这些 Context 用于包装我们调用 addLast 方法时添加的 ChannelHandler（以下简称 handler）。

我们可以分为3个步骤来看编织的过程：

1. 任何一个 ChannelSocket 创建的同时都会创建 一个 pipeline。
2. 当用户或系统内部调用 pipeline 的 add*** 方法添加 handler 时，都会创建一个包装这 handler 的 Context。
3. 这些 Context 在 pipeline 中组成了双向链表。

让我们从代码层面看看他们的编织过程。

**1. Socket 创建的时候创建 pipeline：**
在 SocketChannel 的抽象父类 AbstractChannel 的构造方法中：

![](https://upload-images.jianshu.io/upload_images/4236553-5ffe2d2322b8bdb7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从 newChannelPipeline 方法中获取一个 pipeline，这个方法的标准实现如下：

![](https://upload-images.jianshu.io/upload_images/4236553-fca694d4f6501b7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

创建一个 DefaultChannelPipeline 对象，并传入 channel 对象。这个 DefaultChannelPipeline 是 ChannelPipeline 接口的标准实现。

我们看看他的创建过程：

![](https://upload-images.jianshu.io/upload_images/4236553-58a46fc579e66ad0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 将 channel 赋值给 channel 字段，用于 pipeline 操作 channel。
2. 创建一个 future 和 promise，用于异步回调使用。
3. 创建一个 inbound  的 tailContext，创建一个既是 inbound 类型又是 outbound 类型的 headContext.
4. 最后，将两个 Context 互相连接，形成双向链表。

注意: tailContext 和 HeadContext 非常的重要，所有 pipeline 中的事件都会流经他们，所以我们重点关注 tailContext 和 headContext。

首先看看 TailContext 的设计：一个属于 DefaultChannelPipeline 的内部类。

![](https://upload-images.jianshu.io/upload_images/4236553-16f165c9fe98a42a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

UML 继承图如下：

![UML](https://upload-images.jianshu.io/upload_images/4236553-840ba6dd1ea987c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中可以看出， TailContext 是一个处理入站事件的 handler。

构造方法如下：

````java
      private static final String TAIL_NAME = generateName0(TailContext.class);

      TailContext(DefaultChannelPipeline pipeline) {
            super(pipeline, null, TAIL_NAME, true, false);
            setAddComplete();
        }

    AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                                  boolean inbound, boolean outbound) {
        this.name = ObjectUtil.checkNotNull(name, "name");
        this.pipeline = pipeline;
        this.executor = executor;
        this.inbound = inbound;
        this.outbound = outbound;
        ordered = executor == null || executor instanceof OrderedEventExecutor;
    }
````

从上面的构造方法中可以看出来，Context 果然就是 Context ，囊括了 Channel 所包含的一切，这里说一下 name 是 简单类名+#0 的形式。pipeline 就是当前的 pipeline，executor 是 null，inbound 属性是 true，outbound 属性是 fasle。说明他是一个入站处理器。当有入站事件时，会调用 tailContext。

说完 TailContext ，再看看 HeadContext。

HeadContext 同样时 DefaultChannelPipeline 的内部类，UML 继承图如下：

![](https://upload-images.jianshu.io/upload_images/4236553-232e3b5ef1b90f87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中，可以看出来 HeadContext 非常的全能，既是入站处理器也是出站处理器，任何事件都逃不过他的眼睛。

他的构造方法和 tail 有些许的不同：

````java
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
    setAddComplete();
}

AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                              boolean inbound, boolean outbound) {
    this.name = ObjectUtil.checkNotNull(name, "name");
    this.pipeline = pipeline;
    this.executor = executor;
    this.inbound = inbound;
    this.outbound = outbound;
    ordered = executor == null || executor instanceof OrderedEventExecutor;
}

````

从构造方法上看，唯一的区别就是比 tailContext 多了一个属性 unsafe，而这个属性来自于 pipeline 所属的 channel 的 unsafe，如果大家有印象的话，会记得 channel 初始化的时候，也会初始化一个 unsafe，这个我们今天先不细说，只需要知道他是一个 Netty 中一个直接处理的类，每个类型的 Socket 都有不同的实现。而为什么 head 需要这样一个属性呢？因为 head 需要处理出站数据，还记得出站接口时怎么定义的吗？

![出站处理器定义的方法](https://upload-images.jianshu.io/upload_images/4236553-2d69485554e4ea54.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

出站接口中都是针对数据的操作，比如 read，write，flush 等操作，所以需要 unsafe 这个能够处理数据的工具实例。

为什么 tail 不需要呢？我想你应该知道了，tail 虽然是入站 handler，入站 handler 定义的方法没有需要直接处理数据的，比如 read，write，flush等：

![入站处理器定义的方法](https://upload-images.jianshu.io/upload_images/4236553-4a4d17c32d934c57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

理解这两个处理器的定义很重要，因为每种类型的处理器定义的的任务都是不同的。


**2. 在 add**** 添加处理器的时候创建 Context**

我们看看 DefaultChannelPipeline 的 addLast 方法如何创建的 Context，代码如下：

````java
@Override
public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    for (ChannelHandler h: handlers) {
        if (h == null) {
            break;
        }
        addLast(executor, null, h);
    }
    return this;
}

````

注意，addLast 是个重载方法，你可以选择传入一个线程池，作用是什么呢？当你的业务 handler 非常耗时，甚至阻塞线程，那么 Netty 建议你异步执行该任务，否则将会影响 Netty 的性能。而这个线程池就是用来执行这个 handler 的耗时任务的。

什么时候会返回这个线程池呢？

![](https://upload-images.jianshu.io/upload_images/4236553-1781d82635ae101d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当你调用类似 ChannelActive 方法的时候，会需要 Cotext 的 executor，方法如下：

![](https://upload-images.jianshu.io/upload_images/4236553-ff849ec873cc383f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你没有定义 handler 自己的 executor，那么就使用 channel 的 线程，也就是 IO 线程。你需要十分确定你的业务不会阻塞线程。

再看看 addLast 方法：

````java
@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler);

        newCtx = newContext(group, filterName(name, handler), handler);

        addLast0(newCtx);
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}

````
向 pipeline 添加 handler，参数是线程池，name 是null， handler 是我们或者系统传入的handler。Netty 为了防止多个线程导致安全问题，同步了这段代码，步骤如下：
1. 检查这个 handler 实例是否是共享（Sharable 注解）的，如果不是，并且已经被别的 pipeline 使用了，则抛出异常。
2. 调用 newContext(group, filterName(name, handler), handler) 方法，创建一个 Context。从这里可以看出来了，每次添加一个 handler 都会创建一个关联 Context。
3. 调用 addLast 方法，将 Context 追加到链表中。
4. 如果这个通道还没有注册到 selecor 上，就将这个 Context 添加到这个 pipeline 的待办任务中。当注册好了以后，就会调用 callHandlerAdded0 方法（默认是什么都不做，用户可以实现这个方法）。

我们重点看看第 2 步和第 3 步：
newContext 方法代码如下：

![创建默认的 DefaultChannelHandlerContext 实例](https://upload-images.jianshu.io/upload_images/4236553-16920e97964168f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![构造方法](https://upload-images.jianshu.io/upload_images/4236553-1ed0ae84617fd8a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里的 super 构造方法和 head  tail 一样，没什么不同，有 2 个方法需要注意一下 isInbound 和 isOutbound 方法。这两个方法是辨别这个 handler 是 inbound 还是 outbound 。如果是你，你怎么写？我们还是看看 Netty 是怎么写的吧：

````java
private static boolean isInbound(ChannelHandler handler) {
    return handler instanceof ChannelInboundHandler;
}

private static boolean isOutbound(ChannelHandler handler) {
    return handler instanceof ChannelOutboundHandler;
}
````

很简单，通过 instanceof 关键字判断。哈哈。


再看看第 3 步，如何将这个新创建的 Context 插入到链表中：

![插入链表](https://upload-images.jianshu.io/upload_images/4236553-6f9bc53d9ccb4253.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


也很简单，一个标准的双向链表实现。将新的 Context 的 prev 指向 tail 之前的 prev，将新的 Context 的 next 指向 tail，将 tail 之前的 prev 的 next 指向新的 Context， 将 tail 现在的 prev 指向新的 Context。成功插入到 tail 的前面，所以，这里的 addLast 不是真正的 last，而是除了 tail 的 last，因为 tail 是系统的节点，需要做一些系统工作。


好了，到这里，针对三巨头的创建过程，我们就了解的差不多了，就和我们最初说的一样，每当创建 ChannelSocket 的时候都会创建一个绑定的 pipeline，一对一的关系，同时也创建一个 pipeline，创建 pipeline 的时候也会创建 tail 节点和 head 节点，形成最初的链表。tail 是入站 inbound 类型的 handler，  head 既是 inbound 也是 outbound 类型的 handler。在调用 pipeline 的 addLast 方法的时候，会根据给定的 handler 创建一个 Context，然后，将这个 Context 插入到链表的尾端（tail 前面）。这样，整个三巨头就连接起来了，就能为后面的请求进行流式处理了。

## 3. ChannelPipeline 是如何调度 handler 的

说了这么多，那么当一个请求进来的时候，ChannelPipeline 是如何调用内部的这些 handler 的呢？我们一起来看看。

首先，当一个请求进来的时候，会第一个调用 pipeline 的 相关方法，如果是入站事件，这些方法由 fire 开头，表示开始管道的流动。让后面的 handler 继续处理。

我们看看 DefaultChannelPipeline 是如何实现这些 fire 方法的。

![](https://upload-images.jianshu.io/upload_images/4236553-fb950aae1682ce47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上图中可以看出来，这些方法都是 inbound 的方法，也就是入站事件，调用静态方法传入的也是 inbound 的类型 head handler。这些静态方法则会调用 head 的  ChannelInboundInvoker 接口的方法，再然后调用 handler 的真正方法。

再看看 piepline 的 outbound 的 fire 方法实现：

![](https://upload-images.jianshu.io/upload_images/4236553-ca0c316eeb94acc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

和 inbound 类似，这些都是出站的实现，但是调用的是 outbound 类型的 tail handler 来进行处理，因为这些都是 outbound 事件。

为什么出站是 tail 开始，入站从 head 开始呢？因为出站是从内部外面写，从tail 开始，能够让前面的 handler 进行处理，防止由 handler 被遗漏，比如编码。反之，入站当然是从 head 往内部输入，让后面的 handler 能够处理这些输入的数据。比如解码。

这也解释了虽然 head 也实现了 outbound 接口，但不是从 head 开始执行出站任务。

关于如何调度，请让我用一张图来表示：

![](https://upload-images.jianshu.io/upload_images/4236553-e441911aa953368a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

pipeline 首先会调用 Context 的静态方法 fireXXX，并传入 Context，然后，静态方法调用 Context 的 invoker  方法，而 invoker 方法内部会调用该 Context 所包含的 Handler 的真正的 XXX 方法，调用结束后，如果还需要继续向后传递，就调用 Context 的 fireXXX2 方法，循环往复。

我们将在下一篇文章中详细的解析一个请求在 pipeline 中的流动过程。这幅图仅作抛砖引玉。



好，到这里，关于这三巨头的介绍就差不多了，下面，外面来做一下总结。


## 4. 总结

这是我们 Netty 系列关于 pipeline 的第一篇文章，讲述了关于 pipeline ，Context，Handler 错综复杂的关系，实际上，还是很清晰的。Context 包装 handler，多个 Context 在 pipeline 中形成了双向链表，入站方向叫 inbound，由 head 节点开始，出站方法叫 outbound ，由 tail 节点开始。而节点中间的传递通过  AbstractChannelHandlerContext 类内部的 fire 系列方法，找到当前节点的下一个节点不断的循环传播。是一个完美的过滤器高级形式。


下一篇，将和大家一起在 pipeline 的管道中游走一趟。

good luck！！！！







