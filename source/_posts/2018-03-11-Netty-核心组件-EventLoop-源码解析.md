---
layout: post
title: Netty-核心组件-EventLoop-源码解析
date: 2018-03-11 11:11:11.000000000 +09:00
---
![](https://upload-images.jianshu.io/upload_images/4236553-1d459dc90783b3a2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

在前文 [Netty 启动过程源码分析 （本文超长慎读）(基于4.1.23)](https://www.jianshu.com/p/46861a05ce1e) 中，我们分析了整个服务器端的启动过程。在那篇文章中，我们重点关注了启动过程，而在启动过程中对核心组件并没有进行详细介绍，比如 EventLoop ，Pipeline，Unsafe 等。实际上，Netty 的大部分组件都可以拿出来好好说道，因为每个组件都经过了精心的设计，就像 `《Netty 实战》` 的作者所说：
> Netty 终究是一个框架，他的架构方法和设计原则是：每个小点和它的技术性内容一样重要，穷奇精妙。

**如果仔细看过 Netty ，真的会觉得如作者所说，每个小点都穷奇精妙**。

而今天，我们将分析其最最核心的组件 EventLoop。

## 1. EventLoop 介绍

在上篇文章 [Netty 启动过程源码分析 （本文超长慎读）(基于4.1.23)](https://www.jianshu.com/p/46861a05ce1e) 中，我们基于 Netty 示例代码进行了剖析，其中，第一行代码就是 ：`EventLoopGroup bossGroup = new NioEventLoopGroup(8);` 同时我们也说该组件和核心中的核心。

首先看看 NioEventLoop 的继承图：

![UML 继承图](https://upload-images.jianshu.io/upload_images/4236553-3e7c165ac61a7b12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

使用红框标出了重点部分：
1. ScheduledExecutorService 接口表示是一个定时任务接口，EventLoop 可以接受定时任务。
2. EventLoop 接口：Netty 接口文档说明该接口作用：一旦 Channel 注册了，就处理该Channel对应的所有I/O 操作。
3. SingleThreadEventExecutor 表示这是一个单个线程的线程池。

再来看看 EventLoop 接口：

![](https://upload-images.jianshu.io/upload_images/4236553-6189a7342be0df48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只定义了一个方法，就是parent，返回值类型是 EventLoopGroup。


>注意：一个 EventLoop 将由一个永远都不会改变的Thread 驱动，同时任务（Runnable 或者 Callable）可以直接提交给 EventLoop 实现，以立即执行或者调度执行。根据配置和可用核心的不同，可能会创建多个 EventLoop 实例用以优化资源的使用，并且单个 EventLoop 可能会被指派用于服务多个 Channel。

现在我们可以通俗的解释一下 EventLoop 了，EventLoop 就是一个单例的线程池，里面含有一个死循环的线程不断的做着3件事情：监听端口，处理端口事件，处理队列事件。每个 EventLoop 都可以绑定多个 Channel，而每个 Channel 始终只能由一个 EventLoop 来处理。



## 2. NioEventLoop 的使用 execute 方法

在前面的源码分析中，我们大量的看到了 EventLoop 的使用，一般就是 eventloop.execute(task);
那我就来看 execute 方法的实现：

````java
    @Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {
            addTask(task);
        } else {
            startThread();
            addTask(task);
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
````

首先判断该 EventLoop 的线程是否是当前线程，如果是，直接添加到任务队列中去，如果不是，则尝试启动线程（但由于线程是单个的，因此只能启动一次），随后再将任务添加到队列中去。如果线程已经停止，并且删除任务失败，则执行拒绝策略，默认是抛出异常。

如果 addTaskWakesUp 是 false，并且任务不是 NonWakeupRunnable 类型的，就尝试唤醒 selector。这个时候，阻塞在 selecor 的线程就会立即返回。

我们需要关注一下 addTask 方法：


````java
    protected void addTask(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (!offerTask(task)) {
            reject(task);
        }
    }
````

再看看 offerTask 方法：

````java
    final boolean offerTask(Runnable task) {
        if (isShutdown()) {
            reject();
        }
        return taskQueue.offer(task);
    }

````

注意这个 taskQueue 是什么类型的：

````java
this.maxPendingTasks = Math.max(16, maxPendingTasks);
askQueue = newTaskQueue(this.maxPendingTasks);


 @Override
 protected Queue<Runnable> newTaskQueue(int maxPendingTasks) {
        // This event loop never calls takeTask()
        return maxPendingTasks == Integer.MAX_VALUE ? PlatformDependent.<Runnable>newMpscQueue()
                                                    : PlatformDependent.<Runnable>newMpscQueue(maxPendingTasks);
    }
````

这个 maxPendingTasks 变量默认是 int 最大值，21 亿，所以，后面的默认就是返回 PlatformDependent.<Runnable>newMpscQueue()。

那么这个 PlatformDependent.<Runnable>newMpscQueue() 到底是什么呢？


````java
    /**
     * Create a new {@link Queue} which is safe to use for multiple producers (different threads) and a single
     * consumer (one thread!).
     * @return A MPSC queue which may be unbounded.
     */
    public static <T> Queue<T> newMpscQueue() {
        return Mpsc.newMpscQueue();
    }
````

从注释上看，这是一个可以安全的用于多个生产者（多个线程）和一个单独的消费者（一个线程）的无界队列。这个类不是 Netty 的，而是 Netty 引入的 jctools 包。此队列可以被多个线程写入，但只能有一个线程取出。

我们回到 execute 方法，再看看 startThread() 方法。



## 3.  NioEventLoop  的父类 SingleThreadEventExecutor 的 startThread 方法

当执行 execute 方法的时候，如果当前线程不是 EventLoop 所属线程，则尝试启动线程，也就是 startThread 方法，我们来看该方法：

````java
    private void startThread() {
        if (state == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                try {
                    doStartThread();
                } catch (Throwable cause) {
                    STATE_UPDATER.set(this, ST_NOT_STARTED);
                    PlatformDependent.throwException(cause);
                }
            }
        }
    }
````

该方法首先判断是否启动过了，保证 EventLoop 只有一个线程，如果没有启动过，则尝试使用 Cas 将 state 状态改为 ST_STARTED，也就是已启动。然后调用 doStartThread 方法。如果失败，则进行回滚。

那么就来看 doStartThread 方法：

````java
private void doStartThread() {
    executor.execute(new Runnable() {
        @Override
        public void run() {
            boolean success = false;
            updateLastExecutionTime();
            try {
                SingleThreadEventExecutor.this.run();
                success = true;
            } finally {
                for (;;) {
                    int oldState = state;
                    if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                        break;
                    }
                }
                try {
                    for (;;) {
                        if (confirmShutdown()) {
                            break;
                        }
                    }
                } finally {
                    try {
                        cleanup();
                    } finally {
                        STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                        threadLock.release();
                        terminationFuture.setSuccess(null);
                    }
                }
            }
        }
    });
}
````

楼主精简了部分代码，拆解步骤如下：
1. 首先调用 executor 的 execute 方法，这个 executor 就是在创建 Event LoopGroup 的时候创建的 ThreadPerTaskExecutor 类。该 execute 方法会讲 Runnable 包装成Netty 的 FastThreadLocalThread。该类后面我们将会详细介绍。
2. 任务中，首先判断线程中断状态，然后设置最后一次的执行时间。
3. 执行当前 NioEventLoop 的 run 方法，注意：这个方法是个死循环，是整个 EventLoop 的核心，不然怎么叫 Loop 呢？
4. 在 finally 块中，使用CAS 不断修改 state 状态，改成 ST_SHUTTING_DOWN。也就是当线程 Loop 结束的时候。关闭线程。最后还要死循环确认是否关闭，否则不会 break。然后，执行 cleanup 操作，更新状态为 
ST_TERMINATED，并释放当前线程锁。如果任务队列不是空，则打印队列中还有多少个未完成的任务。并回调 terminationFuture 方法。


从上面的步骤中，我们直到，其实最核心的就是  Event Loop自身的 run 方法。再继续深入 run 方法，一起探究 Loop 到底是什么？



## 4. EventLoop 中的 Loop 到底是什么？

代码如下：
````java
 @Override
    protected void run() {
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.SELECT:
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                    default:
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
````

方法很长，我们拆解一下：
1. 默认的，如果任务队列中有任务，就立即唤醒 selector ，并返回 selector 的 selecotrNow 方法的返回值。如果没有任务，直接返回 -1，这个策略在 DefaultSelectStrategy 中。
2. 如果返回的是 -2， 则继续循环。如果返回的是 -1，也就是没有任务，则调用 selector 的 select 方法，并且设置 wakenUp 为 false。 具体再详细讲。
3. selector 返回后， 当 ioRatio 变量为100的时候（默认50），处理 select 事件，处理完之后执行任务队列中的所有任务。  反之当不是 100 的时候，处理 selecotr 事件，之后给定一个时间内执行任务队列中的任务。可以看到，ioRatio 的作用就是限制执行任务队列的时间。 关于 ioRatio , Netty 是这解释的，在 Netty 中，有2种任务，一种是 IO 任务，一种是非 IO 任务，如果 ioRatio 比例是100 的话，则这个比例无作用。公式则是建立在 IO 时间上的，公式为 ioTime * (100 - ioRatio) / ioRatio ; 也就是说，当 ioRatio 是 10 的时候，IO 任务执行了 100 纳秒，则非IO任务将会执行 900 纳秒，直到没有任务可执行。


从上面的步骤可以看出，整个 run 方法做了3件事情：
1. selector 获取感兴趣的事件。
2. processSelectedKeys 处理事件。
3. runAllTasks 执行队列中的任务。

我们将继续深入这3个方法。

## 5. 核心 select 方法解析

代码如下：


````java
    private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
            for (;;) {
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                if (timeoutMillis <= 0) {
                    if (selectCnt == 0) {// 无任务则超时事件为1秒
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }
                if (hasTasks() && wakenUp.compareAndSet(false, true)) {// 含有任务 && 唤醒 selector 成功； 则立即返回
                    selector.selectNow(); // 立即返回
                    selectCnt = 1;
                    break;
                }

                int selectedKeys = selector.select(timeoutMillis); // 否则阻塞给定时间，默认一秒
                selectCnt ++;
                // 如果1秒后返回，有返回值 || select 被用户唤醒 || 任务队列有任务 || 有定时任务即将被执行； 则跳出循环
                if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                    break;
                }
                if (Thread.interrupted()) {
                    selectCnt = 1;
                    break;
                }
                // 避开 JDK bug
                long time = System.nanoTime();
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    selectCnt = 1;
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&// 没有持续 timeoutMillis 且超过 512次，则认为触发了 JDK 空轮询Bug
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                    // 重建 selector
                    rebuildSelector();
                    selector = this.selector;
                    // 并立即返回
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }
                currentTimeNanos = time;
            }
        } 
    }
````

方法也挺长的，我们来好好拆解该方法：
1. 使用`当前时间`加上`定时任务即将执行的剩余时间（如果没有定时任务，默认1秒）`。得到 selectDeadLineNanos。
2. selectDeadLineNanos 减去当前时间并加上一个缓冲值 0.5秒，得到一个 selecotr 阻塞超时时间。
3. 如果这个值小于1秒，则立即 selecotNow 返回。
4. 如果大于0（默认是1秒），如果任务队列中有任务，并且 CAS 唤醒 selector 能够成功。立即返回。
5. `int selectedKeys = selector.select(timeoutMillis)`，开始真正的阻塞（默认一秒钟），调用的是 SelectedSelectionKeySetSelector 的 select 方法，感兴趣的可以看看该方法。
6. select 方法一秒钟返回后，如果有事件，或者 selector 被唤醒了，或者 任务队列有任务，或者定时任务即将被执行，跳出循环。
7. 如果上述条件不满足，线程被中断了，则跳出循环。
8. **注意**：如果一切正常，开始判断这次 select 的阻塞时候是否大于等于给定的 timeoutMillis 时间，如果没有，且循环了超过 512 次(默认)，则认为触发了 JDK 的 epoll 空轮询 Bug，调用 rebuildSelector 方法重新创建 selector，并 selectorNow 立即返回。

以上9步基本就是 selector 方法的所有。该方法穷奇所有，压榨CPU性能，并避免了 JDK 的 bug。那么，selector 的阻塞时间有哪些地方可以干扰呢？
1. selecotr 返回了事件。
2. 任务队列有任务了。
3. 定时任务即将执行了。
4. 线程被中断了。 
5. 定时任务剩余时间小于 1 秒。
6. 触发了 JDK 的bug。

以上 6 种操作都会让 select 立即返回，不会再这里死循环。

我们继续看看当触发 JDK bug 后重建 selector 的方法。

````java
private void rebuildSelector0() {
    final Selector oldSelector = selector;
    final SelectorTuple newSelectorTuple;

    newSelectorTuple = openSelector();
  
    int nChannels = 0;
    for (SelectionKey key: oldSelector.keys()) {
        Object a = key.attachment();
        int interestOps = key.interestOps();
        key.cancel();
        SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
        if (a instanceof AbstractNioChannel) {
            // Update SelectionKey
            ((AbstractNioChannel) a).selectionKey = newKey;
        }
        nChannels ++;
    }

    selector = newSelectorTuple.selector;
    unwrappedSelector = newSelectorTuple.unwrappedSelector;
}
````

楼主精简了很多异常和日志代码，让我们来拆解一下：
1. 调用 openSelector 方法重新创建一个 Selector，并用 SelectorTuple 包装返回。
2. 循环老的 Selector 的所有注册事件，注册到新的 Selector 上。

这里就是 Netty 解决 JDK bug 的所有核心。


**好了，看完了 selector 方法，我们回到 run 方法。**


## 6. 核心 processSelectedKeys 方法解析

当 selector 返回的时候，我们直到，有可能有事件发生，也有可能是别的原因让他返回了。而处理事件的方法就是 processSelectedKeys，我们进入到该方法查看：

````java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
````
判断 selectedKeys 这个变量，这个变量是一个 Set 类型，但 Netty 内部使用了 SelectionKey 类型的数组，而不是 Map 实现。这个变量什么作用呢？答：当 selector 方法有返回值的时候，JDK 的 Nio 会向这个 set 添加 SelectionKey。通过上面的代码我们看到，如果不是 null（默认开启优化） ，使用优化过的 SelectionKey，也就是数组，如果没有开启优化，则使用 JDK 默认的。

我们看看默认优化的是怎么实现的：

````java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        selectedKeys.keys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            selectedKeys.reset(i + 1);
            selectAgain();
            i = -1;
        }
    }
}

````

该方法还是比较简单的，步骤如下：
1. 循环所有 selectedKeys，拿到该 Key attach 的 Channel，判断是否是 Netty 的 AbstractNioChannel 类型。
2. 如果 needsToSelectAgain 是 true ，则将数组中 start 下标加1 之后的 key 全部设置成null。然后，调用 
selectAgain 方法，该方法会将 needsToSelectAgain 设置成 false，并调用 selectorNow 方法返回。同时也会将循环变量 i 改成 -1，再次重新循环。那么，这个 needsToSelectAgain 默认是 false ，什么时候是 true 呢？答：当调用 cancel 方法的时候，也就是 eventLoop close 的时候，取消这个 key 的事件监听。当取消次数达到了256次，needsToSelectAgain 设置成 true。而这么做的目的是什么呢？结合 Netty 的注释：当 EventLoop close 次数达到 256 次，说明了曾经的 Channel 无效了，Netty 就需要清空数组，方便 GC 回收，然后再次 selectNow ，装填新的 key。


好了，该方法的重点应该是 processSelectedKey 方法，而判断则是 a instanceof AbstractNioChannel ，还记得 Channel 注册的时候吗：

![AbstractNioChannel doRegister 方法 ](https://upload-images.jianshu.io/upload_images/4236553-78bf24285a1c4020.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上面的代码中可以看出，Netty 会将 Channel 绑定到 key 上，然后在循环到事件处理的时候，拿出来直接使用。

那我们就看看 processSelectedKey 内部逻辑：


````java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();// NioMessageUnsafe
    if (!k.isValid()) {
        final EventLoop eventLoop = ch.eventLoop();
        unsafe.close(unsafe.voidPromise());
        return;
    }

    int readyOps = k.readyOps();
    if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
        int ops = k.interestOps();
        ops &= ~SelectionKey.OP_CONNECT;
        k.interestOps(ops);
        unsafe.finishConnect();
    }
    .
    if ((readyOps & SelectionKey.OP_WRITE) != 0) {
        ch.unsafe().forceFlush();

    }
    if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
        unsafe.read();
    }
}
````

看到这里的代码，相信大家肯定很亲切，这不就是 Nio 的标准做法吗？

注意：这里的 unsafe 是每个 key 所对应的 Channel 对应的 unsafe。因此处理逻辑也是不同的。

可以说，run 方法中的 processSelectedKeys 方法的核心就是，拿到 selector 返回的所有 key 进行循环调用 processSelectedKey 方法， processSelectedKey 方法中会调用每个 Channel 的 unsafe 的对应方法。



好了， processSelectedKeys 方法到此为止。








## 7. 核心 runAllTasks 解析

再看看 runAllTasks ，Task 里面都是一些非 IO 任务。就是通过 execute 提交的那些任务，都会添加的 task 中。

代码如下：

````java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }

    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task);

        runTasks ++;

        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}   

````

我们来拆解一下该方法：
1. 将定时任务队列（PriorityQueue 类型的 scheduledTaskQueue）中即将执行的任务都添加到普通的 Mpsc 队列中。
2. 从 Mpsc 队列中取出任务，如果是空的，则执行 tailTasks（Mpsc 无界队列） 中的任务，然后直接结束该方法。
3. 如果不是空，则进入死循环，跳出条件有2个，1是给定的时间到了，2是没有任务了。有一个需要注意的地方就是，这个时间的检查不是每一次都检查的，而是64次循环执行一次检查，因为获取纳米时间的开销较大。
4. 最后执行 tailTasks 中的任务，并更新 lastExecutionTime 最后执行时间。

从上面的分析看出，有3种队列，分别是普通的 taskQueue，定时任务的 scheduledTaskQueue， Mpsc 的 tailQueue。

1. taskQueue，这个我们很熟悉，基本上，我们现在遇到的都是执行 execute 方法，通过 addTask 方法添加进去的。
2. 定时任务的 scheduledTaskQueue，通常在第一次调用 schedule 方法的时候会创建这个队列。同时这个任务是一般是调用 schedule 方法的时候添加进去的。主要当然是定时任务，同时也是异步的，会返回一个 ScheduledFuture 异步对象。这个对象可以添加监听器或者做一些回调，类似 permise。刚刚在上面也说了，这个队列是一个优先级队列，那么这个队列的优先级是怎么比较的呢？他的默认比较器是 SCHEDULED_FUTURE_TASK_COMPARATOR 对象，比较策略是调用 ScheduledFutureTask 的 compareTo 方法，首先任务队列的剩余时间，然后比较 id，每个任务创建时都会生成一个唯一ID，也就是加入时间的顺序。在每次 poll 之后，都会比较所有的任务，让优先级最高的任务排在数组第一位。

3. tailQueue 针对这个任务，Netty 在 executeAfterEventLoopIteration 方法的注释上意思是，添加一个任务，在 EnentLoop 下个周期运行，就像我们源码中的，每次在运行任务之后，如果还有时间的话就会运行这个队列中的任务，一般这个队列中放一些优先级不高的任务。但楼主在源码中没有找到应用他的地方。多说一句，该 Queue 也是一个 Mpsc 队列。


最后对 lastExecutionTime  进行赋值，有什么作用呢？在 confirmShutdown 方法中，会对该变量进行判断：

![判断](https://upload-images.jianshu.io/upload_images/4236553-b95093bb91f820ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![通常该方法都在死循环中](https://upload-images.jianshu.io/upload_images/4236553-8f367362b57a6cce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在 EventLoop 的父类 SingleThreadEventExecutor 的 doStartThread 方法的 finally 块中，也就是如果 run 方法结束了，会执行这里的逻辑，确认是否关闭了，如果定时任务最后一次的执行时间距离现在的时间 小于等于 `优雅关闭的静默期时间（默认2秒）`，则唤醒 selector，并睡眠 0.1 秒，返回 false，表示还没有关闭呢？并继续循环，在 confirmShutdown 的上方逻辑上回继续调用 runAllTasks 方法。此处应该时担心关闭的时候还有尚未完成的定时任务吧。


好，到这里，关于 runAllTasks 方法就解释的差不多了。

## 总结

总上面的分析中，我们看到了 EventLoop 作为 Netty 的核心是如何处理，每次执行 ececute 方法都是向队列中添加任务。当第一次添加时就回启动线程，执行 run 方法，而 run 方法是整个 EventLoop 的核心，就像 EventLoop 的名字一样，Loop Loop ，不停的 Loop ，Loop 做什么呢？做3件事情。
1. 调用 selecotr 的 select 方法，默认阻塞一秒钟，如果有定时任务，则在定时任务剩余时间的基础上在加上0.5秒进行阻塞。当执行 execute 方法的时候，也就是添加任务的时候，回唤醒 selecor，防止 selecotr 阻塞时间过长。
2. 当 selector 返回的时候，回调用 processSelectedKeys 方法对 selectKey 进行处理。
3. 当 processSelectedKeys 方法执行结束后，则按照 iaRatio 的比例执行 runAllTasks 方法，默认是 IO 任务时间和非 IO 任务时间是相同的，你也可以根据你的应用特点进行调优 。比如 非 IO 任务比较多，那么你就讲 ioRatio 调小一点，这样非 IO 任务就能执行的长一点。防止队列钟积攒过多的任务。

好，关于 EventLoop 的分析就到这里，水平不高，能力有限，有疏忽的地方请指出。

good luck ！！！！




