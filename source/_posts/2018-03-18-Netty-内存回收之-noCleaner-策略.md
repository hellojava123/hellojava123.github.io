---
layout: post
title: Netty-内存回收之-noCleaner-策略
date: 2018-03-18 11:11:11.000000000 +09:00
---
![](https://upload-images.jianshu.io/upload_images/4236553-d1c7aa1b05f873a2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

对于堆外内存，使用 System.gc() 是不靠谱的，依赖老年代 FGC 也是不靠谱的，而且大部分调优指南都设置了 -DisableExplicitGC 禁用 System.gc()。所以主动回收比较靠谱， JDK 在 DirectByteBuffer 中提供了 Cleaner 用来主动释放内存。同时还有 Unsafe 的 freeMemory 方法也可以。 下面看看他是怎么做的。这里以非池化创建直接内存为例。


##  UnpooledByteBufAllocator  newDirectBuffer 方法

代码如下：

````java
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

关键点在于 noCleaner 的结果。影响其结果的代码如下：

````java
// 构造方法，
public UnpooledByteBufAllocator(boolean preferDirect, boolean disableLeakDetector, boolean tryNoCleaner) {
    super(preferDirect);
    this.disableLeakDetector = disableLeakDetector;
    noCleaner = tryNoCleaner && PlatformDependent.hasUnsafe()
            && PlatformDependent.hasDirectBufferNoCleanerConstructor();
}
// tryNoCleaner 结果来自 PlatformDependent.useDirectBufferNoCleaner()
public UnpooledByteBufAllocator(boolean preferDirect, boolean disableLeakDetector) {
    this(preferDirect, disableLeakDetector, PlatformDependent.useDirectBufferNoCleaner());
}
// 判断是否含有 DirectByteBuffer 构造器，有则 true
public static boolean useDirectBufferNoCleaner() {
    return USE_DIRECT_BUFFER_NO_CLEANER;
}

// 根据是否含有 DirectByteBuffer 的构造器判断，如果没有，USE_DIRECT_BUFFER_NO_CLEANER=false
if (maxDirectMemory == 0 || !hasUnsafe() || !PlatformDependent0.hasDirectBufferNoCleanerConstructor()) {
    USE_DIRECT_BUFFER_NO_CLEANER = false;
    DIRECT_MEMORY_COUNTER = null;
} else {
    USE_DIRECT_BUFFER_NO_CLEANER = true;
    if (maxDirectMemory < 0) {
        maxDirectMemory = maxDirectMemory0();
        if (maxDirectMemory <= 0) {
            DIRECT_MEMORY_COUNTER = null;
        } else {
            DIRECT_MEMORY_COUNTER = new AtomicLong();
        }
    } else {
        DIRECT_MEMORY_COUNTER = new AtomicLong();
    }
}



// 获取 DirectByteBuffer 的构造器
final ByteBuffer direct;
Constructor<?> directBufferConstructor;
long address = -1;
try {
    final Object maybeDirectBufferConstructor =
            AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    try {
                        final Constructor<?> constructor =
                                direct.getClass().getDeclaredConstructor(long.class, int.class);
                        Throwable cause = ReflectionUtil.trySetAccessible(constructor, true);
                        if (cause != null) {
                            return cause;
                        }
                        return constructor;
                    } catch (NoSuchMethodException e) {
                        return e;
                    } catch (SecurityException e) {
                        return e;
                    }
                }
            });

            ((Constructor<?>) maybeDirectBufferConstructor).newInstance(address, 1);
            directBufferConstructor = (Constructor<?>) maybeDirectBufferConstructor;
         
    
} finally {
    if (address != -1) {
        UNSAFE.freeMemory(address);
    }
}
DIRECT_BUFFER_CONSTRUCTOR = directBufferConstructor;

````

noCleaner  为 true：创建 InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf 对象。简称 noCleaner；
noCleaner  为false：创建 InstrumentedUnpooledUnsafeDirectByteBuf 对象。简称 hasCleaner；



**两者构造器方式不同：**
> noCleaner 反射调用 private DirectByteBuffer(long addr, int cap)
hasCleaner new 操作调用 DirectByteBuffer(int cap)

**两个释放内存方式不同:**
> noCleaner  使用 unSafe.freeMemory(address);
hasCleaner  使用 DirectByteBuffer 的 Cleaner 的 clean 方法。


**hasCleaner   的 clean 方法有 2 种策略：**
>  1.Java9 使用 Unsafe 的 invokeCleaner 方法。调用了 ByteBuffer 的 Cleaner 的 clean 方法。
    2. Java6 --- Java9 使用 DirectByteBuffer 的 属性 Cleaner 的 clean 方法。

**clean 方法原理:**

> 这个 clean 方法内部调用了一个名为 thunk 的 Deallocator 线程的 run 方法。该线程对象在创建 DirectByteBuffer 的时候同时创建。该线程的 run 方法内部会调用 unsafe 的 freeMemory 方法，同时还会调用 Bits.unreserveMemory 方法，该方法会相应的减小已经使用的内存大小数字（因为，每次申请直接内存都需要 Bits 判断是否足够，如果 FGC 后还不够，OOM，所以，这里的做法还是挺重要的）

>注意：这个 Cleaner 是个虚引用，DirectByteBuffer  创建他的时候，会将自己放入虚引用的构造函数中，如果这个 DirectByteBuffer  被回收了（无人再引用这个 Cleaner），那么 GC 将会把这个 Cleaner 赋值给 Reference 的 pending 变量中，专门有一条 ReferenceHandler 的线程会死循环执行 Reference 的 tryHandlePending 方法，这个方法会调用 pending 的 clean 方法，完成内存回收操作。


**这是 cleaner 对象的构造时机：**

````java
DirectByteBuffer(int cap) {                   // package-private

    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    // 这里构造 cleaner
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}

`````

**该构造只有一个 int 参数**


所以，你知道了吧，noCleaner 的构造方法是不能调用 cleaner 的 clean 方法的。只能使用 unSafe 的 freeMemory 方法。而这就是 Netty 默认的做法。

同时，noCleaner  的构造方法也没有向 Bits 申请内存的内容，在申请内存的时候，性能会比 hasCleaner  要好一点。关于 Bits 的设计，我觉得不够优雅。当内存不够了，就 System.gc()，却只休眠 100 毫秒。根本不够回收到堆外内存。

实际上，Cleaner 的作用除了更新一下 Bits 的一些属性，方便下次申请内存之外，别无作用。

我猜想 Netty 使用 noCleaner  是性能优化的考虑吧。为了防止用户忘记使用 ReferenceCountUtil.release（）， 导致内存泄漏，Netty 还使用了虚引用跟踪每一个 ByteBuf，基本上避免了内存泄漏的发生。

综上所述：noCleaner  无论是在申请内存还是释放内存都比使用 hasCleaner  性能好要好一点。

































