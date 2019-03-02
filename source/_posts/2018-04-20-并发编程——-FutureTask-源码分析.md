---
layout: post
title: 并发编程——-FutureTask-源码分析
date: 2018-04-20 11:11:11.000000000 +09:00
---
## 1. 前言

当我们在 Java 中使用异步编程的时候，大部分时候，我们都会使用 Future，并且使用线程池的 submit 方法提交一个 Callable 对象。然后调用 Future 的 get 方法等待返回值。而 FutureTask 是 Future 的一个实现，也是我们今天的主角。

我们就从源码层面分析 FutureTask.

## 2. FutureTask 初体验

我们一般接触的都是 Future ，而不是 FutureTask , Future 是一个接口， FutureTask 是一个标准的实现。在我们向线程池提交任务的时候，线程池会创建一个 FutureTask 返回。

```java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

newTaskFor 方法就是创建一个了一个 FutureTask 返回。

```java
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
```


而线程池就会执行 FutureTask 的 run 方法。

那么，我们看看 FutureTask 的 UML。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-d998d1b07d607c56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





可以看出，FutureTask 实现了 Runnable，Future 。Runnable 就不必说了，一个 run 方法，那 Future 呢？

```java
boolean cancel(boolean mayInterruptIfRunning);
boolean isCancelled();
boolean isDone();
V get() throws InterruptedException, ExecutionException;
V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;   
``` 

主要是这 5 个方法撑起了 Future，功能相对而言比较薄弱，毕竟这只是一个 Future ，而不是 Promise。

FutureTask 还有一个内部类，WaitNode ，结构如下：

```java
static final class WaitNode {
    volatile Thread thread;
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```

看起来是不是和 AQS 的节点似曾相识呢？

FutureTask 内部维护了一个栈结构，和 AQS 的队列有所区别。

实际上，在之前的版本中，FutureTask 确实直接使用的 AQS ，但是 Doug lea 又对该类进行了优化，优化的目的是 ：

> 主要是为了避免有些用户在取消竞争期间保留中断状态。


而内部依然使用了一个 volatile 的 state 变量来控制状态，同时使用了一个栈结构来保存等待的线程。

**至于原因，当然是 FutureTask 的 get 方法是支持并发的，多个线程可以获取到同一个 FutureTask 的同一个结果，而这些线程在 get 的阻塞过程中必然是要挂起自己等待的。**

知道了 FutureTask 的结构。我们知道，线程池肯定会执行 FutureTask 的 run 方法，所以，我们到他的 run 方法看看。

同时，我们也要看看关键方法 —— get 方法。



## 3. FutureTask 的 get 方法

代码如下：
```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

首先判断状态，然后挂起自己等待，最后，返回结果，代码很简单。

注意：FutureTask 中有 7 种状态：

```java
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

构造时，状态就是 NEW，当任务完成中，状态变成 COMPLETING。当任务彻底完成，状态变成 NORMAL。

我们重点看看 awaitDone 和 report 方法。

awaitDone 方法代码：

```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
            LockSupport.park(this);
    }
}
```

上面的方法相对于 JUC 其他的类，还是比较简单的。需要注意一个点：get 方法是可以并发访问的，当并发访问的时候，需要将这些线程保存在 FutureTask 内部的栈中。

 简单说说方法步骤：
1. 如果线程中断了，删除节点，并抛出异常。
2. 如果字体大于  COMPLETING ，说明任务完成了，返回结果。
3. 如果等于 COMPLETING，说明任务快要完成了，自旋一会。
4. 如果 q 是 null，说明这是第一次进入，创建一个新的节点。保存当前线程引用。
5. 如果还没有修改过 waiters 变量，就使用 CAS  修改当前 waiters 为当前节点，这里是一个栈的结构。
6. 根据时间策略挂起当前线程。
7. 当线程醒来后，继续上面的判断，正常情况下，返回数据。

再看看 report 方法：

```java
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

也还是很简单的，拿到结果，判断状态，如果状态正常，就返回值，如果不正常，就抛出异常。

总结一下 get 方法：

FutureTask 通过挂起自己等待异步线程唤醒，然后拿去异步线程设置好的数据。


## 4. FutureTask 的 run 方法

上面总结说，FutureTask 通过挂起自己等待异步线程唤醒，然后拿去异步线程设置好的数据。

那么这个过程在哪里呢？答案就是在 run 方法里。我们知道，线程池在执行 FutureTask 的时候，肯定会执行他的 run 方法。所以，我们看看他的 run 方法：

```java
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

方法逻辑如下：
1. 判断状态。
2. 执行 callable 的 call 方法。
3. 设置结果并唤醒等待的所有线程。

看看 set 方法是如何设置结果的：

```java
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
```

先将状态变成 COMPLETING，然后设置结果，再然后设置状态为 NORMAL，最后执行 finishCompletion 方法唤醒等待线程。

finishCompletion 代码如下：

```java 
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
```

该方法先将 waiters 修改成 null，然后遍历栈中所有节点，也就是所有等待的线程，依次唤醒他们。

最后执行 done 方法。这个方法是留个子类扩展的。FutureTask 中是个空方法。比如 Spring 的 ListenableFutureTask 就扩展了该方法。还有 JUC 里的 QueueingFuture 类也扩展了该方法。

如果异常了就将状态改为 EXCEPTIONAL。

如果用户执行了 cancel（true）方法。该方法 Java doc 如下：

> 试图取消对此任务的执行。如果任务已完成、或已取消，或者由于某些其他原因而无法取消，则此尝试将失败。当调用 cancel 时，如果调用成功，而此任务尚未启动，则此任务将永不运行。如果任务已经启动，则 mayInterruptIfRunning 参数确定是否应该以试图停止任务的方式来中断执行此任务的线程。

也就是说，这个 mayInterruptIfRunning 决定当任务已经在执行了，还要终止这个任务。如果 mayInterruptIfRunning 是 true ，就会先将状态改成 INTERRUPTING，然后调用线程的 interrupt 方法，最后，设置状态为 INTERRUPTED。

在 run 方法的 finally 块中，对 INTERRUPTING 有判断，也就是说，在 INTERRUPTING 和 INTERRUPTED 的这段时间，会执行 finally 块，那么这个时候，就需要自旋等待状态变成 INTERRUPTED。

具体代码如下：

```java
private void handlePossibleCancellationInterrupt(int s) {
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt
}
```


## 5. 总结

关于 FutureTask 就介绍完了，该类最重要的就是 get 方法和 run 方法，run 方法负责执行 callable 的 call 方法并设置返回值到一个变量中， get 方法负责阻塞直到 run 方法执行完毕任务唤醒他，然后 get 方法回去结果。

同时，FutureTask 为了多线程可以并发调用 get 方法，使用了一个栈结构保存所有等待的线程。也就是说，所有的线程都等得到 get 方法的结果。

虽然 FutureTask 的设计很好，但我仍然觉得使用异步是更好的选择，效率更高。
