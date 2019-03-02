---
layout: post
title: 并发编程-——-ScheduledThreadPoolExecutor
date: 2018-04-21 11:11:11.000000000 +09:00
---
## 1. 前言

在前面的文章中，我们介绍了定时任务类 Timer ，他是 JDK 1.3 中出现的，位于 java.util 包下。而今天说的 `ScheduledThreadPoolExecutor `的是在 JUC 包下，是 JDK1.5 新增的。

今天就来说说这个类。

## 2. API 介绍

该类内部结构和 `Timer `还是有点类似的，也是  3 个类：

* `ScheduledThreadPoolExecutor`：程序员使用的接口。
* `DelayedWorkQueue` ： 存储任务的队列。
* `ScheduledFutureTask` ： 执行任务的线程。


**构造方法介绍**：

```java
// 使用给定核心池大小创建一个新 ScheduledThreadPoolExecutor。
ScheduledThreadPoolExecutor(int corePoolSize)  
// 使用给定初始参数创建一个新 ScheduledThreadPoolExecutor。
ScheduledThreadPoolExecutor(int corePoolSize, RejectedExecutionHandler handler)  
// 使用给定的初始参数创建一个新 ScheduledThreadPoolExecutor。
ScheduledThreadPoolExecutor(int corePoolSize, ThreadFactory threadFactory)  
// 使用给定初始参数创建一个新 ScheduledThreadPoolExecutor。
ScheduledThreadPoolExecutor(int corePoolSize, ThreadFactory threadFactory, RejectedExecutionHandler handler)  
```

`ScheduledThreadPoolExecutor `最多支持 3 个参数：核心线程数量，线程工厂，拒绝策略。

为什么没有最大线程数量？由于 `ScheduledThreadPoolExecutor` 内部是个无界队列，`maximumPoolSize` 也就没有意思了。


再介绍一下他的 API 方法，请原谅我将 JDK 文档照抄过来了，就当是备忘吧，如下：

```java
protected <V> RunnableScheduledFuture<V> decorateTask(Callable<V> callable, RunnableScheduledFuture<V> task) // 修改或替换用于执行 callable 的任务。
  
protected <V> RunnableScheduledFuture<V> decorateTask(Runnable runnable, RunnableScheduledFuture<V> task) // 修改或替换用于执行 runnable 的任务。        
    
void execute(Runnable command) // 使用所要求的零延迟执行命令。  

boolean	getContinueExistingPeriodicTasksAfterShutdownPolicy() // 获取有关在此执行程序已 shutdown 的情况下、是否继续执行现有定期任务的策略。      
   
boolean	getExecuteExistingDelayedTasksAfterShutdownPolicy() // 获取有关在此执行程序已 shutdown 的情况下是否继续执行现有延迟任务的策略。  

BlockingQueue<Runnable>	getQueue() // 返回此执行程序使用的任务队列。     

boolean	remove(Runnable task) // 从执行程序的内部队列中移除此任务（如果存在），从而如果尚未开始，则其不再运行。     
  
<V> ScheduledFuture<V>	schedule(Callable<V> callable, long delay, TimeUnit unit) // 创建并执行在给定延迟后启用的 ScheduledFuture。    
    
ScheduledFuture<?>	schedule(Runnable command, long delay, TimeUnit unit) // 创建并执行在给定延迟后启用的一次性操作。  
     
ScheduledFuture<?>	scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) // 创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期；也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推。 
     
ScheduledFuture<?>	scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) // 创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。  
   
void setContinueExistingPeriodicTasksAfterShutdownPolicy(boolean value) // 设置有关在此执行程序已 shutdown 的情况下是否继续执行现有定期任务的策略。  
   
void setExecuteExistingDelayedTasksAfterShutdownPolicy(boolean value) // 设置有关在此执行程序已 shutdown 的情况下是否继续执行现有延迟任务的策略。  
      
void shutdown() // 在以前已提交任务的执行中发起一个有序的关闭，但是不接受新任务。
    
List<Runnable>	shutdownNow() // 尝试停止所有正在执行的任务、暂停等待任务的处理，并返回等待执行的任务列表。  
     
<T> Future<T> submit(Callable<T> task) //  提交一个返回值的任务用于执行，返回一个表示任务的未决结果的 Future。 
   
Future<?> submit(Runnable task) //  提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future。  
     
<T> Future<T> submit(Runnable task, T result) //  提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future。
```


最经常使用的几个方法如下：

```java
// 使用给定核心池大小创建一个新 ScheduledThreadPoolExecutor。
ScheduledThreadPoolExecutor(int corePoolSize)  

// 创建并执行在给定延迟后启用的一次性操作。  
ScheduledFuture<?>	schedule(Runnable command, long delay, TimeUnit unit) 
  
// 创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期；也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推。 
ScheduledFuture<?>	scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) 

// 创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。       
ScheduledFuture<?>	scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) 
```


除了默认的构造方法，还有 3 个 `schedule` 方法。我们将分析他们内部的实现。

##  3. 构造方法内部实现

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

我们感兴趣的就是这个 `DelayedWorkQueue` 队列了。他也是一个阻塞队列。这个队列的数据结构是堆。同时，这个 `queue` 也是可比较的，比较什么呢？任务必须实现 `compareTo` 方法，这个方法的比较逻辑是：比较任务的执行时间，如果任务的执行时间相同，则比较任务的加入时间。

因此，`ScheduledFutureTask` 有 2 个变量：
* `time` ： 任务的执行时间。
* `sequenceNumber`：任务的加入时间。

这两个变量就是用来比较任务的执行顺序的。整个调度的顺序就是这个逻辑。


## 4. 几个 schedule  方法的的区别

刚刚说了，有 3 个 `schedule` 方法：
1. `ScheduledFuture<?>  schedule(Runnable command, long delay, TimeUnit unit) `
创建并执行在给定延迟后启用的一次性操作。  

2. `ScheduledFuture<?>  scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)  `
创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期；也就是将在` initialDelay` 后开始执行，然后在 `initialDelay+period `后执行，接着在` initialDelay + 2 * period `后执行，依此类推。 

3. `ScheduledFuture<?>  scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) `
创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。       

第一个方法执行在给定的时间后，执行一次就结束。


**有点意思的地方是 第二个方法和 第三个方法，他们直接的区别。**

这两个方法都可以重复的调用。但是，重复调用的逻辑有所区别，这里就是比 Timer 好用的地方。

他们的共同点在于：**必须等待上个任务执行完毕才能执行下个任务。**

不同点在于：**他们调度的时间粗略是不同的。**

`scheduleAtFixedRate` 方法的执行周期都是固定的，也就是，他是以上一个任务的开始执行时间作为起点，加上之后的 period 时间，调度下次任务。

`scheduleWithFixedDelay` 方法则是以上一个任务的结束时间作为起点，加上之后的 period 时间，调度下次任务。

有什么区别呢？

如何任务执行时间很短，那就没上面区别。但是，如果任务执行时间很长，超过了 period 时间，那么区别就出来了。


我们假设一下。

我们设置 `period` 时间为  2 秒，而任务耗时 5 秒。

这个两个方法的区别就体现出来了。

`scheduleAtFixedRate` 方法将会在上一个任务结束完毕立刻执行，他和上一个任务的开始执行时间的间隔是 **5** 秒（因为必须等待上一个任务执行完毕）。

`scheduleWithFixedDelay` 方法将会在上一个任务结束后，注意：**再等待 2 秒，**才开始执行，那么他和上一个任务的开始执行时间的间隔是 **7** 秒。

所以，我们在使用 `ScheduledThreadPoolExecutor` 的过程中需要注意任务的执行时间不能超过间隔时间，如果超过了，最好使用` scheduleAtFixedRate` 方法，防止任务堆积。

当然，也和具体的业务有关。不能一概而论。但一定要注意这两个方法的区别。




## 5. scheduled 方法的实现

我们看看 scheduleAtFixedRate 方法的内部实现。

```java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay,
                                              long period,
                                              TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(period));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}
```

创建一个 `ScheduledFutureTask` 对象，然后装饰一个这个` Future` ，该类实现是直接返回，子类可以有自己的实现，在这个任务外装饰一层。


然后执行 ` delayedExecute` 方法，最后返回 `Future`。

这个 `ScheduledFutureTask` 实现了很多接口，比如 `Future`，`Runnable`， `Comparable`，` Delayed` 等。

`ScheduledFutureTask` 的构造方法如下：

```java
ScheduledFutureTask(Runnable r, V result, long ns, long period) {
    super(r, result);
    this.time = ns;
    this.period = period;
    this.sequenceNumber = sequencer.getAndIncrement();
}

public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}

public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}

static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
````

层层递进，该类首先通过一个原子静态 `int `对象这只任务的入队编号，然后创建一个 `Callable`，这个 Callable 是一个适配器，适配了` Runnable `和` Callable`，也就是将` Runnable `包装成 `callabe`， 他的 `call `方法就是调用给定任务的 run 方法。当然，这里的 `result `是没有什么作用的。

如果你传递的是一个  `callable` ，那么，就调用 `FutureTask` 的 `run `方法，设置真正的返回值。

这里使用了适配器模式，还是挺有趣的。

总的来说，这个 `ScheduledFutureTask` 基于 `FutureTask`， 关于 `FutureTask` 我们之前从源码介绍过了。

而他自己重写了几个方法：`compareTo`， `getDelay`， `run`，`isPeriodic `4 个方法。

我们还是要看看 `delayedExecute` 的实现。

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        // 添加进队列。
        super.getQueue().add(task);
        // 如果线程池关闭了，且不可以在当前状态下运行任务，且从队列删除任务成功，就给任务打上取消标记。
        // 第二个判断是由两个变量控制的（下面是默认值）：
        // continueExistingPeriodicTasksAfterShutdown = false 表示关闭的时候应取消周期性任务。默认关闭
        // executeExistingDelayedTasksAfterShutdown = true。表示关闭的时候应取消非周期性的任务。默认不关闭。
        // running 状态下，canRunInCurrentRunState 必定返回 ture。
        // 非 running 状态下，canRunInCurrentRunState 根据上面的两个值返回。
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            // 开始执行任务
            ensurePrestart();
    }
}
```

说说上面的方法。
1. 判断是否关闭，关闭则拒绝任务。
2. 如果不是 关闭状态，则添加进队列，而添加队列的顺序我们之前讲过了，根据 `ScheduledFutureTask` 的 `compareTo` 方法来的，先比较执行时间，再比较添加顺序。
3. 如果这个过程中线程池关闭了，则判断此时是否应该取消任务，根据两个变量来的，注释里面写了。默认的策略是，如果是周期性的任务，就取消，反之不取消。
4. 如果没有关闭线程池。就调用线程池里的线程执行任务。

整体的过程如图：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-7cbe02aef6b00d45.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


注意上面的图，如果是周期性的任务，则会在执行完毕后，归还队列。

从哪里可以看出来呢？

`ScheduledFutureTask` 的 `run` 方法：

```java
public void run() {
    // 是否是周期性任务
    boolean periodic = isPeriodic();
    // 如果不可以在当前状态下运行，就取消任务（将这个任务的状态设置为CANCELLED）。
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    // 如果不是周期性的任务，调用 FutureTask # run 方法
    else if (!periodic)
        ScheduledFutureTask.super.run();
    // 如果是周期性的。
    // 执行任务，但不设置返回值，成功后返回 true。（callable 不可以重复执行）
    else if (ScheduledFutureTask.super.runAndReset()) {
        // 设置下次执行时间
        setNextRunTime();
        // 再次将任务添加到队列中
        reExecutePeriodic(outerTask);
    }
}
```

逻辑如下：

1. 如果不能再当前状态下运行了，就取消这个任务。
2. 如果不是周期性的任务，就执行 `FutureTask` 的 `run` 方法。
3. 如果是周期性的任务，就需要执行 `runAndReset `方法。
4. 执行完毕后，重写设置当前任务的下次执行时间，然后添加进队列中。

而管理整个执行过程的就是` ScheduledThreadPoolExecutor `的父类 `ThreadPoolExecutor `的` runWorker `方法。其中，该方法会从队列中取出数据，也就是调用队列的 `take` 方法。

关于` DelayedWorkQueue` 的` take `方法，其中有个 `leader` 变量，如果 leader 不是空，说明已经有线程在等待了，那就阻塞当前线程，如果是空，说明，队列的第一个元素已经被更新了，就设置当前线程为 `leader`.

这是一个 `Leader-Follower ` 模式，Doug Lea 说的。

当然，`take `方法整体的逻辑还是不变的。从队列的头部拿数据。使用 `Condition` 做线程之间的协调。

## 5. 总结

关于 `ScheduledThreadPoolExecutor` 调度类，我们分析的差不多了，总结一下。

`ScheduledThreadPoolExecutor` 是个定时任务线程池，类似 `Timer`，但是比 `Timer `强大，健壮。

比如不会像 Timer 那样，任务异常了，整个调度系统就彻底无用了。

也比` Timer` 多了` Rate` 模式（`Rate 和 Delay`）。

这两种模式的区别就是任务执行的起点时间不同，`Rate `是从上一个任务的开始执行时间开始计算；`Delay` 是从上一个任务的结束时间开始计算。

因此，如果任务本身的时间超过了间隔时间，那么这两种模式的间隔时间将会不一致。

而任务的排序是通过 `ScheduledFutureTask `的 `compareTo `方法排序的，规则是先比较执行时间，如果时间相同，再比较加入时间。

还要注意一点就是：如果任务执行过程中异常了，那么将不会再次重复执行。因为 `ScheduledFutureTask `的 `run `方法没有做` catch `处理。所以程序员需要手动处理，相对于 Timer 异常就直接费了调度系统来说，要好很多。

`ScheduledThreadPoolExecutor` 的是实现基于 `ThreadPoolExecutor`，大部分功能都是重用的父类，**只是自己在执行完毕之后，重新设置时间，并再次将任务还到了队列中**，形成了定时任务。






