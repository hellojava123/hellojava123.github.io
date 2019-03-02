---
layout: post
title: 并发编程之-LinkedBolckingQueue-源码剖析
date: 2018-01-08 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-b86798c8d8b7d1d8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




## 前言

JDK 1.5 之后，Doug Lea 大神为我们写了很多的工具，整个 concurrent 包基本都是他写的。也为我们程序员写好了很多工具，包括我们之前说的线程池，重入锁，线程协作工具，ConcurrentHashMap 等等，今天我们要讲的是和 ConcurrentHashMap 类似的数据结构，LinkedBolckingQueue，阻塞队列。在生产者消费者模型中，该类可以帮助我们快速的实现业务功能。

1. 如何使用？
2. 源码分析



## 1. 如何使用？

我们在生产者消费者模型，生产者向一个数据共享通道存放数据，消费者从相同的数据共享通道获取数据，将生产和消费完全隔离，不仅是生产者消费者，现在流行的消息队列，比如各种MQ，kafka，和这个都差不多。废话不多说，直接来个demo ，看看怎么使用：

```java
  public static void main(String[] args) {
    LinkedBlockingQueue linkedBlockingQueue = new LinkedBlockingQueue(1024);
    for (int i = 0; i < 5; i++) {
      final int num = i;
      new Thread(() -> {
        try {
          for (int j = 0; ; j++) {
            linkedBlockingQueue.put(num + "号线程的" + j + "号商品");
            Thread.sleep(5000);
          }
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }).start();
    }

    for (int i = 0; i < 5; i++) {
      new Thread(() -> {
        try {
          for (; ; ) {
            System.out.println("消费了" + linkedBlockingQueue.take());
            Thread.sleep(1000);
          }
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }).start();
    }
  }


```

运行结果：

```
消费了0号线程的0号商品
消费了3号线程的0号商品
消费了2号线程的0号商品
消费了1号线程的0号商品
消费了4号线程的0号商品
消费了2号线程的1号商品
消费了1号线程的1号商品
消费了0号线程的1号商品
消费了3号线程的1号商品
消费了4号线程的1号商品
消费了1号线程的2号商品
消费了0号线程的2号商品
消费了2号线程的2号商品
消费了3号线程的2号商品
消费了4号线程的2号商品
·········
```

从上面的代码中，我们使用了5条线程分别向队列中插入数据，也就是一个字符串，然后让5个线程从队列中取出数据并打印，可以看到，生产者插入的数据从消费者线程中被打印，没有漏掉一个。

注意，这里的 put 方法和 take 方法都是阻塞的，不然就不是阻塞队列了，什么意思呢？如果队列满了，put 方法就会等待，直到队列有空为止，因此该方法使用时需要注意，如果业务即时性很高，那么最好使用带有超时选项的 offer （V，long，TimeUnit），方法，同样， take 方法也是如此，当队列中没有的时候，就会阻塞，直到队列中有数据为止。同样可以使用 poll（long, TimeUnit）方法超时退出。

当然不止这几个方法，楼主将常用的方法总结一下：

插入方法：
```java
 
    // 如果满了，立即返回false
    boolean b = linkedBlockingQueue.offer("");
    // 如果满了，则等待到给定的时间，如果还满，则返回false
    boolean b2 = linkedBlockingQueue.offer("", 1000, TimeUnit.MILLISECONDS);
    // 阻塞直到插入为止
    linkedBlockingQueue.put("");
```

取出方法：

```java
    // 如果队列为空，直接返回null
    Object o3 = linkedBlockingQueue.poll();
    // 如果队列为空，一直阻塞到给定的时间
    Object o1 = linkedBlockingQueue.poll(1000, TimeUnit.MILLISECONDS);
    // 阻塞，直到取出数据
    Object o = linkedBlockingQueue.take();
    // 获取但不移除此队列的头；如果此队列为空，则返回 null。
    Object peek = linkedBlockingQueue.peek();

```

那么这些方法内部是如何实现的呢？


## 2. 源码分析

阻塞队列，重点看 put 阻塞方法和 take 阻塞方法。

##### put 方法：

```java
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
```

该方法步骤如下：
1. 根据给定的值创建一个 Node 对象，该对象有2个属性，一个是 item，一个是 Node 类型的 next，是链表结构的节点。
2. 获取 put 的锁，注意，这里，put 锁和 take 锁是分开的。也就是说，当你插入的时候和取出的时候用的不是一把锁，可以高效并发，但是如果两个线程同时插入就会阻塞。
3. 获取链表的长度。
4. 使用中断锁，如果调用了线程的中断方法，那么，处于阻塞中的线程就会抛出异常。
5. 判断如果当前链表长度达到了设置的长度，默认是 int 最大型，就调用 put 锁的伙伴 Condition 对象 notFull 让当前线程挂起等待。 直到 take 方法中会调用 notFull 对象的 signal 方法唤醒。
6. 调用 enqueue 方法，将刚刚创建的 Node 节点连接到链表上。
7. 将链表长度变量 count 加一。 判断如果加一后，链表长度还小于链表规定的容量，那么就唤醒其他等待在 notFull 对象上的线程，告诉他们可以取数据了。
8. 放开锁，让其他线程争夺锁（非公平锁）。
9. 如果c是0，表示队列已经有一个数据了，通知唤醒挂在 notEmpty 的线程，告诉他们可以取数据了。


##### take 方法如下：

```java
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }

```
步骤如下：
1. 获取链长度，获取 take 锁。
2. 调用可中断的 lock 方法。开始锁住。
3. 如果队列是空，则挂起线程。开始等待。
4. 如果不为空，则调用 dequeue 方法，拿到头节点的数据，并将头节点更新。
5. 将队列长度减一。判断如果队列长度大于1，通知等待在 notEmpty 上的线程，可以拿数据了。
6. 解锁。
7. 如果变量 c 和 容量相同，而刚刚又消费了一个节点，说明队列不满了，则通知生产者可以添加数据了。
8. 返回数据。


##### boolean offer(E e, long timeout, TimeUnit unit)  源码：

```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }

```

该方法会阻塞给定的时间，如果时间到了，则返回false。
和 put 方法很相似，步骤如下：
1. 将时间转成纳秒。
2. 获取 put 锁。
3. 调用可中断锁方法。
4. 如果容量满了，并且设置的等待时间小于0，返回 false，表示插入失败，反之，调用 notFull 方法等待给定的时间，并返回一个负数，当第二次循环的时候，继续判断，如果还是满的并且小于0，返回false。
5. 如果容量没有满，或者等待过程被唤醒，则调用 enqueue 插入数据。
6. 获取当前链表长度。
7. 判断链表长度+1是否小于设置的容量。如果小于，则链表没有满，通知生产者可以添加数据了。
8. 释放锁。 如果 c 等于 0，表示之前没有数据，但是现在已经加入一个数据了，可以通知其他的消费者来消费了。
9. 返回 true。


##### E poll(long timeout, TimeUnit unit) 源码分析

```java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E x = null;
        int c = -1;
        long nanos = unit.toNanos(timeout);
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

该方法会阻塞给定的时间，如果取不到数据，返回null。

步骤其实和上面的差不多，楼主偷个懒，就不解释了。


## 总结

从源码分析中，我们可以看到，整个阻塞队列就是由重入锁和Condition 组合实现的，和我们之前用 synchronized  加上 wait 和 notify 实现很相似，只是楼主的那个例子没有使用队列，因此无法将锁分开，也就是我们之前说的锁分离的技术。那么，整体的性能当然不能和 Doug Lea 大神的比了。

好了。今天的并发源码分析，就到这里。

good luck！！！！











