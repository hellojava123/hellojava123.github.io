---
layout: post
title: Netty-源码剖析之-unSafe-read-方法
date: 2018-03-18 11:11:11.000000000 +09:00
---
![](https://upload-images.jianshu.io/upload_images/4236553-e55f46526d13babd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


目录：
1. NioSocketChannel$NioSocketChannelUnsafe 的 read 方法
2. 首先看 ByteBufAllocator
3. 再看 RecvByteBufAllocator.Handle
4. 两者如何配合进行内存分配
5. 如何读取到 ByteBuf
6. 总结

## 前言

在之前的文章 [Netty 核心组件 Pipeline 源码分析（二）一个请求的 pipeline 之旅](https://www.jianshu.com/p/9c1925e24e95)中，我们知道了当客户端请求进来的时候，boss 线程会将 Socket 包装后交给 worker 线程，worker 线程会将这个 Socket 注册 selector  的读事件，当读事件进来的时候，会调用 unsafe 的 read 方法，这个方法的主要作用是读取 Socket 缓冲区的内存，并包装成 Netty 的 ByteBuf 对象，最后传递进 pipeline 中的所有节点完成处理。

今天，我们就要好好的看看这个 read方法的实现。


## 1. NioSocketChannel$NioSocketChannelUnsafe 的 read 方法

源码如下：

````java
public final void read() {
    final ChannelConfig config = config();
    final ChannelPipeline pipeline = pipeline();
    // 用来处理内存的分配:池化或者非池化 UnpooledByteBufAllocator
    final ByteBufAllocator allocator = config.getAllocator();
    // 用来计算此次读循环应该分配多少内存 AdaptiveRecvByteBufAllocator 自适应计算缓冲分配
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
    allocHandle.reset(config);// 重置为0

    ByteBuf byteBuf = null;
    boolean close = false;
    try {
        do {
            byteBuf = allocHandle.allocate(allocator);
            allocHandle.lastBytesRead(doReadBytes(byteBuf));
            if (allocHandle.lastBytesRead() <= 0) {// 如果上一次读到的字节数小于等于0，清理引用和跳出循环
                // nothing was read. release the buffer.
                byteBuf.release();// 引用 -1
                byteBuf = null;
                close = allocHandle.lastBytesRead() < 0;// 如果远程已经关闭连接
                if (close) {
                    // There is nothing left to read as we received an EOF.
                    readPending = false;
                }
                break;
            }

            allocHandle.incMessagesRead(1);//  totalMessages += amt;
            readPending = false;
            pipeline.fireChannelRead(byteBuf);
            byteBuf = null;
        } while (allocHandle.continueReading());

        allocHandle.readComplete();
        pipeline.fireChannelReadComplete();

        if (close) {
            closeOnRead(pipeline);
        }
    } catch (Throwable t) {
        handleReadException(pipeline, byteBuf, t, close, allocHandle);
    } finally {
        if (!readPending && !config.isAutoRead()) {
            removeReadOp();
        }
    }
}
````

代码很长，怎么办呢？当然是拆解，然后逐个击破。步骤如下：
1. 获取到 Channel 的 config 对象，并从该对象中获取内存分配器，还有"计算内存分配器"。
2. 将 `计算内存分配器` 重置。
3. 进入一个循环，循环体的作用是：使用内存分配器获取数据容器-----ByteBuf，调用 doReadBytes 方法将数据读取到容器中，如果这次读取什么都没有或远程连接关闭，则跳出循环。还有，如果满足了跳出推荐，也要结束循环，不能无限循环，默认16 次，默认参数来自 AbstractNioByteChannel 的 属性 ChannelMetadata 类型的 METADATA 实例。每读取一次就调用 pipeline 的 channelRead 方法，为什么呢？因为由于 TCP 传输如果包过大的话，丢失的风险会更大，导致重传，所以，大的数据流会分成多次传输。而 channelRead 方法也会被调用多次，因此，使用 channelRead 方法的时候需要注意，如果数据量大，最好将数据放入到缓存中，读取完毕后，再进行处理。
4. 跳出循环后，调用 `allocHandle` 的 readComplete 方法，表示读取已完成，并记录读取记录，用于下次分配合理内存。
5. 调用 pipeline 的方法。


接下来就一步步看。

## 2. 首先看 ByteBufAllocator

首先看看这个节点的定义：

>   Implementations are responsible to allocate buffers. Implementations of this interface are expected to be hread-safe.
实现负责分配缓冲区。这个接口的实现应该是线程安全的。


![定义了如上方法](https://upload-images.jianshu.io/upload_images/4236553-34d07961151497cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过这个接口，可以看出来，主要作用是创建 ByteBuf，这个 ByteBuf 是 Netty 用来替代 NIO 的 ByteBuffer 的，是存储数据的缓存区。其中，这个接口有一个默认实现  ByteBufUtil.DEFAULT_ALLOCATOR ：该实现根据配置创建一个 池化或非池化的缓存区分配器。该参数是 `io.netty.allocator.type`。

同时，由于很多方法都是重载的，那就说说上面的主要方法作用：
````java
buffer() // 返回一个 ByteBuf 对象，默认直接内存。如果平台不支持，返回堆内存。
heapBuffer（）// 返回堆内存缓存区
directBuffer（）// 返回直接内存缓冲区
compositeBuffer（） // 返回一个复合缓冲区。可能同时包含堆内存和直接内存。
ioBuffer（） // 当当支持 Unsafe 时，返回直接内存的 Bytebuf，否则返回返回基于堆内存，当使用 PreferHeapByteBufAllocator 时返回堆内存
````








## 3. 再看 RecvByteBufAllocator.Handle

首先看这个接口：

![](https://upload-images.jianshu.io/upload_images/4236553-f6f18e4fac080d3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中， Handle 是 RecvByteBufAllocator 的内部接口。而 RecvByteBufAllocator 是如何定义的呢？

>  Creates a new handle.  The handle provides the actual operations and keeps the internal information which is required for predicting an optimal buffer capacity.
创建一个新的句柄。句柄提供了实际操作，并保留了用于预测最佳缓冲区容量所需的内部信息。

该接口只定义了一个方法：newHandle（）。

而 handle 的作用是什么呢？

````java

ByteBuf allocate(ByteBufAllocator alloc);//创建一个新的接收缓冲区，其容量可能大到足以读取所有入站数据和小到数据足够不浪费它的空间。
int guess();// 猜测所需的缓冲区大小，不进行实际的分配
void reset(ChannelConfig config);// 每次开始读循环之前，重置相关属性
void incMessagesRead(int numMessages);// 增加本地读循环的次数
void lastBytesRead(int bytes); // 设置最后一次读到的字节数
int lastBytesRead(); // 最后一次读到的字节数
void attemptedBytesRead(int bytes); // 设置读操作尝试读取的字节数
void attemptedBytesRead(); // 获取尝试读取的字节数
boolean continueReading(); // 判断是否需要继续读
void readComplete(); // 读结束后调用
````
从上面的方法中，可以看出，该接口的主要作用就是计算字节数，如同 RecvByteBufAllocator 的文档说的那样，根据预测和计算最佳大小的缓存区，确保不浪费。



## 4. 两者如何配合进行内存分配

在默认的 config （NioSocketChannelConfig）中，allocator 来自 ByteBufAllocator 接口的默认实例，allocHandle 来自 AdaptiveRecvByteBufAllocator 自适应循环缓存分配器 的内部类 HandleImpl。

好，知道了他们的默认实现，我们一个方法看看。

首先看 reset 方法：
````java
public void reset(ChannelConfig config) {
    this.config = config;
    maxMessagePerRead = maxMessagesPerRead();
    totalMessages = totalBytesRead = 0;
}
````
设置了上次获取的最大消息读取次数（默认16），将之前计算的读取消息总数归零。该方法如同他的名字，归零重置。

再看看 allocHandle.allocate(allocator) 方法的实现。

````java
public ByteBuf allocate(ByteBufAllocator alloc) {
	return alloc.ioBuffer(guess());
}
````

我们刚刚说的 ioBuffer 方法，该方法默认返回直接内存缓冲区。而 guess() 方法返回一个猜测的大小，一个 nextReceiveBufferSize 属性，默认 1024，也就是说，默认创建一个 1024 大小的直接内存缓冲区。这个值的设定来自 HandleImpl 的构造方法，存储在一个 SIZE_TABLE 的数组中。

我们还是看看 RecvByteBufAllocator 的实现类 AdaptiveRecvByteBufAllocator 的具体内容吧

````java
static final int DEFAULT_MINIMUM = 64; // 缓存区最小值
static final int DEFAULT_INITIAL = 1024; // 缓冲区初始值
static final int DEFAULT_MAXIMUM = 65536; // 缓冲区最大值

private static final int INDEX_INCREMENT = 4;// 当发现缓存过小，数组下标自增值
private static final int INDEX_DECREMENT = 1;// 当发现缓冲区过大，数组下标自减值

private static final int[] SIZE_TABLE;

static {
    List<Integer> sizeTable = new ArrayList<Integer>();
    for (int i = 16; i < 512; i += 16) {
        sizeTable.add(i);
    }

    for (int i = 512; i > 0; i <<= 1) {
        sizeTable.add(i);
    }

    SIZE_TABLE = new int[sizeTable.size()];
    for (int i = 0; i < SIZE_TABLE.length; i ++) {
        SIZE_TABLE[i] = sizeTable.get(i);
    }
}
````
楼主在上面的代码中写了注释，这个 SIZE_TABLE 的作用是存储缓存区大小的一个 int 数组，从 static 块中可以看到，这个数组从16开始，同时递增16，直到值到了 512，也就是下标 31 的地方，递增策略变为了 每次 * 2，直到溢出。最终的数组长度为 53。而对应的值接近 int 最大值。

好，回到 allocate 方法中，进入到 ioBuffer 方法查看：

````java
public ByteBuf ioBuffer(int initialCapacity) {
    if (PlatformDependent.hasUnsafe()) {
        return directBuffer(initialCapacity);
    }
    return heapBuffer(initialCapacity);
}
````

判断，如果平台支持 unSafe，就使用直接内存，否则使用堆内存，初始大小就是我们刚刚说的 1024。而这个判断的标准是：如果尝试获取 Unsafe 的时候有异常了，则赋值给一个  UNSAFE_UNAVAILABILITY_CAUSE 对象，否则赋值为 null，Netty 通过这个判 Null 确认平台是否支持 Unsafe。


我们继续看看 directBuffer 方法的实现：

````java

public ByteBuf directBuffer(int initialCapacity) {
    return directBuffer(initialCapacity, DEFAULT_MAX_CAPACITY);
}
// 
public ByteBuf directBuffer(int initialCapacity, int maxCapacity) {
    if (initialCapacity == 0 && maxCapacity == 0) {
        return emptyBuf;
    }
    validate(initialCapacity, maxCapacity);
    return newDirectBuffer(initialCapacity, maxCapacity);
}
// 
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
    final ByteBuf buf;
    if (PlatformDependent.hasUnsafe()) {
        buf = noCleaner ? new InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(this, initialCapacity, maxCapacity) :
                new InstrumentedUnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
    } else {
        buf = new InstrumentedUnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
    }
    return disableLeakDetector ? buf : toLeakAwareBuffer(buf);
}

````
由于方法层层递进，楼主将代码合在一起，最终调用的是 newDirectBuffer，根据 noCleaner 参数决定创建一个 ByteBuf，这个属性怎么来的呢？当 unsafe 不是 null 的时候，会尝试获取 DirectByteBuffer 的构造器，如果成功获取，则 noCleaner 属性为 true。

这个 noCleaner  属性的详细介绍请看这里[Netty 内存回收之 noCleaner 策略](https://www.jianshu.com/p/5b0395d513fc).

默认情况下就是 true，那么，也就是创建了一个 InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf 对象，该对象构造如下:

````java
@1
InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf(
        UnpooledByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(alloc, initialCapacity, maxCapacity);
}

@2
UnpooledUnsafeNoCleanerDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(alloc, initialCapacity, maxCapacity);
}

@3
public UnpooledUnsafeDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);
    if (alloc == null) {
        throw new NullPointerException("alloc");
    }
    if (initialCapacity < 0) {
        throw new IllegalArgumentException("initialCapacity: " + initialCapacity);
    }
    if (maxCapacity < 0) {
        throw new IllegalArgumentException("maxCapacity: " + maxCapacity);
    }
    if (initialCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
    }

    this.alloc = alloc;
    setByteBuffer(allocateDirect(initialCapacity), false);
}

@4
static ByteBuffer newDirectBuffer(long address, int capacity) {
    ObjectUtil.checkPositiveOrZero(capacity, "capacity");
    return (ByteBuffer) DIRECT_BUFFER_CONSTRUCTOR.newInstance(address, capacity);
}

````

最终使用了 DirectByteBuffer  的构造器进行反射创建。而这个构造器是没有默认的 new 创建的 Cleaner 对象的。因此称为 noCleaner。

创建完毕后，调用 setByteBuffer ，将这个 DirectByteBuffer  包装一下。

回到 newDirectBuffer 方法。

最后根据 disableLeakDetector 属性判断释放进行自动内存回收（也就是当你忘记回收的时候，帮你回收），原理这里简单的说一下，使用虚引用进行跟踪。 FastThreadLocal 的内存回收类似。我们将在以后的文章中详细说明此策略。

到这里，创建 ByteBuf 的过程就结束了。

可以说，大部分工作都是 allocator 做的，allocHandle 的作用就是提供了如何分配一个合理的内存的策略。

## 5. 如何读取到 ByteBuf

回到 read 方法，doReadBytes(byteBuf) 就是将 Channel 的内容读取到容器中，并返回一个读取到的字节数。

代码如下：

````java
protected int doReadBytes(ByteBuf byteBuf) throws Exception {
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.attemptedBytesRead(byteBuf.writableBytes());
    return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
}
````
获取到 `内存预估器`，设置一个 attemptedBytesRead 属性为 ByteBuf 的可写字节数。这个参数可用于后面分配内存时的一些考量。

然后调用 byteBuf.writeBytes（）方法。传入了 NIO 的 channel，还有刚刚的可写字节数。进入到该方法查看：

````java
@1
public int writeBytes(ScatteringByteChannel in, int length) throws IOException {
    ensureWritable(length);
    int writtenBytes = setBytes(writerIndex, in, length);
    if (writtenBytes > 0) {
        writerIndex += writtenBytes;
    }
    return writtenBytes;
}

````

首先对长度进行校验，确保可写长度大于0，如果被并发了导致容量不够，将这个底层的 ByteBuffer 的容量增加传入的长度。

关于 ByteBuf 的 wirteIndex ，如下图：

![](http://upload-images.jianshu.io/upload_images/4236553-db077a19e426423e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

回到 writeBytes 方法，调用 setBytes 方法，将流中输入写入到缓冲区。方法如下：

````java
public int setBytes(int index, ScatteringByteChannel in, int length) throws IOException {
    ensureAccessible();
    ByteBuffer tmpBuf = internalNioBuffer();
    tmpBuf.clear().position(index).limit(index + length);
    try {
        return in.read(tmpBuf);
    } catch (ClosedChannelException ignored) {
        return -1;
    }
}
````

非常熟悉的 NIO 操作。

首先获取到内部 ByteBuffer 的共享缓冲区，赋值给临时的 tmpNioBuf 属性。然后返回这个引用。将这个引用清空，并将指针移动到给定 index 为止，然后 limit 方法设置缓存区大小。

最后调用 Channel 的 read 方法，将Channel 数据读入到 ByteBuffer 中。读的过程时线程安全的，内部使用了 synchronized 关键字控制写入 buffer 的过程。返回了读到的字节数。


回到 writeBytes 方法，得到字节数之后，将这个字节数追加到 writerIndex 属性，表示可写字节变小了。


回到 read 方法。allocHandle 得到读取到的字节数，调用 lastBytesRead 方法，该方法的作用时调整下一次分配内存的大小。进入到该方法查看：

````java
public void lastBytesRead(int bytes) {
    // If we read as much as we asked for we should check if we need to ramp up the size of our next guess.
    // This helps adjust more quickly when large amounts of data is pending and can avoid going back to
    // the selector to check for more data. Going back to the selector can add significant latency for large
    // data transfers.
    if (bytes == attemptedBytesRead()) {
        record(bytes);
    }
    super.lastBytesRead(bytes);
}

````

Netty 写了注释：
> 如果我们读的内容和我们要求的一样多，我们应该检查一下是否需要增加下一个猜测的大小。
这有助于在等待大量数据时更快地进行调整，并且可以避免返回选择器以检查更多数据。回到选择器可以为大型数据传输添加显著的延迟。

当获取的字节数和预估的一样大，则需要进行扩容。看看 record 方法实现：

````java
private void record(int actualReadBytes) {
    if (actualReadBytes <= SIZE_TABLE[max(0, index - INDEX_DECREMENT - 1)]) {
        if (decreaseNow) {
            index = max(index - INDEX_DECREMENT, minIndex);
            nextReceiveBufferSize = SIZE_TABLE[index];
            decreaseNow = false;
        } else {
            decreaseNow = true;
        }
    } else if (actualReadBytes >= nextReceiveBufferSize) {
        index = min(index + INDEX_INCREMENT, maxIndex);
        nextReceiveBufferSize = SIZE_TABLE[index];
        decreaseNow = false;
    }
}
````

如果实际读取到的字节数小于等于预估的字节 下标 - 2（排除2以下），则将容量缩小一个下标。如果实际读取到的字节数大于等于预估的。则将下标增加 4，下次创建的 Buffer 容量也相应增加。如果不满足这两个条件，什么都不做。


回答 lastBytesRead 方法，该方法记录了读取到的总字节数并且更新了最后一次的读取字节数。总字节数会用来判断是否可以结束读取循环。如果什么都没有读到，将最多持续到读 16（默认） 次。


回到 read 方法。

如果最后一次读取到字节数小于等于0，跳出循环，不做 channelRead 操作。
反之，将 totalMessages 加1，这个就是用来记录循环次数，判断不能超过 16次。

调用 fireChannelRead 方法，方法结束后，将这个 Buffer 的引用置为null，

判断是否需要继续读取，带入如下：

````java
public boolean continueReading(UncheckedBooleanSupplier maybeMoreDataSupplier) {
    return config.isAutoRead() &&
           (!respectMaybeMoreData || maybeMoreDataSupplier.get()) &&
           totalMessages < maxMessagePerRead &&
           totalBytesRead > 0;
}
````
几个条件：
1. 首先是否自动读取。
2. 且猜测是否还有更多数据，如果实际读取的和预估的一致，说明可能还有数据没读，需要再次循环。
3. 如果读取次数为达到 16 次，继续读取。
4. 如果读取到的总数大于0，说明有数据，继续读取。


这里的循环的主要原因就像我们刚刚说的，TCP 传输过大数据容易丢包（带宽限制），因此会将大包分好几次传输，还有就是可能预估的缓冲区不够大，没有充分读取 Channel 的内容。



## 6. 总结

从 `NioSocketChannel$NioSocketChannelUnsafe` 的实现看 read 方法。每个 ByteBuf 都会由一个 Config 实例中的 ByteBufAllocator 对象创建，池化或非池化，直接内存或堆内存，这些都根据系统是否支持或参数设置，底层使用的是 NIO 的 API。今天我们看的是非池化的直接内存。同时，为了节省内存，为每个 ByteBufAllocator  配置了一个 handle，用于计算和预估缓冲区大小。

还有一个需要注意的地方就是 noCleaner 策略。这是 Netty 的一个优化。针对默认的直接内存创建和销毁做了优化--------不使用 JDK 的 cleaner 策略。

最终读取数据到封装了 NIO ByteBuffer 实例的 Netty 的 ByteBuf 中，其中，如果数据量超过 1024，则会读取超过两次，但最多不超过 16 次， 这个次数可以设置，也就是说，可能会调用超过2次 fireChannelRead 方法，使用的时候需要注意（存起来一起在 ChannelReadComplete 使用之类的方法）。

好，关于 Netty 读取 Socket 数据到容器中的逻辑，就到这里。

good luck！！！























