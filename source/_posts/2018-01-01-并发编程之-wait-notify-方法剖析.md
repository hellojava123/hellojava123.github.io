---
layout: post
title: 并发编程之-wait-notify-方法剖析
date: 2018-01-01 11:11:11.000000000 +09:00
---

![](http://upload-images.jianshu.io/upload_images/4236553-b9c1ca6e42fc1af8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

2018 元旦快乐。

摘要：

1. notify wait 如何使用？
2. 为什么必须在同步块中？
3. 使用 notify wait 实现一个简单的生产者消费者模型
4. 底层实现原理 

## 1. notify wait 如何使用？

今天我们要学习或者说分析的是 Object 类中的 wait notify 这两个方法，其实说是两个方法，这两个方法包括他们的重载方法一共有5个，而Object 类中一共才 12 个方法，可见这2个方法的重要性。我们先看看 JDK 中的代码：

```java
public final native void notify();

public final native void notifyAll();
 
public final void wait() throws InterruptedException {
    wait(0);
}

public final native void wait(long timeout) throws InterruptedException;

public final void wait(long timeout, int nanos) throws InterruptedException {
    if (timeout < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
                            "nanosecond timeout value out of range");
    }

    if (nanos > 0) {
        timeout++;
    }

    wait(timeout);
}
```

就是这五个方法。其中有3个方法是 native 的，也就是由虚拟机本地的c代码执行的。有2个 wait 重载方法最终还是调用了 wait（long） 方法。

首先还是 know how。来一个最简单的例子，看看如何使用这两个方法。

```java
package cn.think.in.java.two;

import java.util.concurrent.TimeUnit;

public class WaitNotify {

  final static Object lock = new Object();

  public static void main(String[] args) {

    new Thread(new Runnable() {
      @Override
      public void run() {
        System.out.println("线程 A 等待拿锁");
        synchronized (lock) {
          try {
            System.out.println("线程 A 拿到锁了");
            TimeUnit.SECONDS.sleep(1);
            System.out.println("线程 A 开始等待并放弃锁");
            lock.wait();
            System.out.println("被通知可以继续执行 则 继续运行至结束");
          } catch (InterruptedException e) {
          }
        }
      }
    }, "线程 A").start();

    new Thread(new Runnable() {
      @Override
      public void run() {
        System.out.println("线程 B 等待锁");
        synchronized (lock) {
          System.out.println("线程 B 拿到锁了");
          try {
            TimeUnit.SECONDS.sleep(5);
          } catch (InterruptedException e) {
          }
          lock.notify();
          System.out.println("线程 B 随机通知 Lock 对象的某个线程");
        }
      }
    }, "线程 B").start();
  }


}

```

运行结果：
> 线程 A 等待拿锁
线程 B 等待锁
线程 A 拿到锁了
线程 A 开始等待并放弃锁
线程 B 拿到锁了
线程 B 随机通知 Lock 对象的某个线程
被通知可以继续执行 则 继续运行至结束


在上面的代码中，线程 A 和 B 都会抢这个 lock 对象的锁，A 的运气比较好（也可能使 B 拿到锁），他先拿到了锁，然后调用了 wait 方法，放弃了锁，并挂起了自己，这个时候等待锁的 B 就拿到了锁，然后通知了A，但是请注意，通知完毕之后，B 线程并没有执行完同步代码块中的代码，因此，A 还是拿不到锁的，因此无法运行，等到B线程执行完毕，出了同步块，这个时候 A 线程才被激活得以继续执行。

使用 wait 方法和 notify 方法可以使 2 个无关的线程进行通信。也就是面试题中常提到的线程之间如何通信。

如果没有 wait 方法和 noitfy 方法，我们如何让两个线程通信呢？简单的办法就是让某个线程循环去检查某个标记变量，比如：

```java
while （value != flag） {
  Thread.sleep(1000);
}
doSomeing();
```
上面的这段代码在条件不满足使就睡眠一段时间，这样做到目的是防止过快的”无效尝试“，这种方式看似能够实现所需的功能，但是却存在如下问题：

1. 难以确保及时性。因为等待的1000时间会导致时间差。
2. 难以降低开销，如果确保了及时性，休眠时间缩短，将大大消耗CPU。

但是有了Java 自带的 wait 方法 和 notify 方法，一切迎刃而解。官方说法是等待/通知机制。一个线程在等待，另一个线程可以通知这个线程，实现了线程之间的通信。


## 2. 为什么必须在同步块中？

注意，这两个方法的使用必须是在 synchroized 同步块中，并且在当前对象的同步块中，如果在 A 对象的方法中调用 B 对象的 wait 或者 notify 方法，虚拟机会抛出 IllegalMonitorStateException，非法的监视器异常，因为你这个线程持有的监视器和你调用的监视器的不是一个对象。

那么为什么这两个方法一定要在同步块中呢？

这里要说一个专业名词：竞态条件。什么是竞太条件呢？

> 当两个线程竞争同一资源时，如果对资源的访问顺序敏感，就称存在竞态条件。

竞态条件会导致程序在并发情况下出现一些bugs。多线程对一些资源的竞争的时候就会产生竞态条件，如果首先要执行的程序竞争失败排到后面执行了，那么整个程序就会出现一些不确定的bugs。这种bugs很难发现而且会重复出现，这是因为线程间会随机竞争。


假设有2个线程，分别是生产者和消费者，他们有各自的任务。

1.1生产者检查条件（如缓存满了）-> 1.2生产者必须等待
2.1消费者消费了一个单位的缓存 -> 2.2重新设置了条件（如缓存没满） -> 2.3调用notifyAll()唤醒生产者

我们希望的顺序是： 1.1->1.2->2.1->2.2->2.3
但是由于CPU执行是随机的，可能会导致 2.3 先执行，1.2 后执行，这样就会导致生产者永远也醒不过来了！

所以我们必须对流程进行管理，也就是同步，通过在同步块中并结合 wait 和 notify 方法，我们可以手动对线程的执行顺序进行调整。


## 3. 使用 notify wait 实现一个简单的生产者消费者模型

虽然很多书中都不建议我们直接使用 notify 和 wait 方法进行并发编程，但仍然需要我们重点掌握。楼主写了一个简单的生产者消费者例子：

#### 简单的缓存类：

```java

public class Queue {

  final int num;
  final List<String> list;
  boolean isFull = false;
  boolean isEmpty = true;


  public Queue(int num) {
    this.num = num;
    this.list = new ArrayList<>();
  }


  public synchronized void put(String value) {
    try {
      if (isFull) {
        System.out.println("putThread 暂停了，让出了锁");
        this.wait();
        System.out.println("putThread 被唤醒了，拿到了锁");
      }

      list.add(value);
      System.out.println("putThread 放入了" + value);
      if (list.size() >= num) {
        isFull = true;
      }
      if (isEmpty) {
        isEmpty = false;
        System.out.println("putThread 通知 getThread");
        this.notify();
      }
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

  public synchronized String get(int index) {
    try {
      if (isEmpty) {
        System.err.println("getThread 暂停了，并让出了锁");
        this.wait();
        System.err.println("getThread 被唤醒了，拿到了锁");
      }

      String value = list.get(index);
      System.err.println("getThread 获取到了" + value);
      list.remove(index);

      Random random = new Random();
      int randomInt = random.nextInt(5);
      if (randomInt == 1) {
        System.err.println("随机数等于1， 清空集合");
        list.clear();
      }

      if (getSize() < num) {
        if (getSize() == 0) {
          isEmpty = true;
        }
        if (isFull) {
          isFull = false;
          System.err.println("getThread 通知 putThread 可以添加了");
          Thread.sleep(10);
          this.notify();
        }
      }
      return value;


    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    return null;
  }


  public int getSize() {
    return list.size();
  }


````

#### 生产者线程：

```java
class PutThread implements Runnable {

  Queue queue;

  public PutThread(Queue queue) {
    this.queue = queue;
  }

  @Override
  public void run() {
    int i = 0;
    for (; ; ) {
      i++;
      queue.put(i + "号");

    }
  }
}

```

#### 消费者线程：

```java
class GetThread implements Runnable {

  Queue queue;

  public GetThread(Queue queue) {
    this.queue = queue;
  }

  @Override
  public void run() {
    for (; ; ) {
      for (int i = 0; i < queue.getSize(); i++) {
        try {
          Thread.sleep(1000);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        String value = queue.get(i);

      }
    }
  }
}

```

大家有兴趣可以跑跑看，能够加深这两个方法的理解，实际上，JDK 内部的阻塞队列也是类似这种实现，但是，不是用的 synchronized ，而是使用的重入锁。

基本上经典的生产者消费者模式的有着如下规则：

等待方遵循如下规则：
1. 获取对象的锁。
2. 如果条件不满足，那么调用对象的 wait 方法，被通知后仍要检查条件。
3. 条件满足则执行相应的逻辑。

对应的伪代码入下：
```java
synchroize( 对象 ){
    while(条件不满足){
      对象.wait();
    }
    对应的处理逻辑......
}
```

通知方遵循如下规则：
1. 获得对象的锁。
2. 改变条件。
3. 通知所有等待在对象上的线程。

对应的伪代码如下：
```java
synchronized（对象）{
  改变条件
  对象.notifyAll();
}
```

## 4. 底层实现原理 


知道了如何使用，就得知道他的原理到底是什么？

首先我们看，使用这两个方法的顺序一般是什么？

1. 使用 wait ，notify 和 notifyAll 时需要先对调用对象加锁。
2. 调用 wait 方法后，线程状态有 Running 变为 Waiting，并将当前线程放置到对象的 **等待队列**。
3. notify 或者 notifyAll 方法调用后， 等待线程依旧不会从 wait 返回，需要调用 noitfy 的线程释放锁之后，等待线程才有机会从 wait 返回。
4. notify 方法将等待队列的一个等待线程从等待队列种移到**同步队列**中，而 notifyAll 方法则是将**等待队列**种所有的线程全部移到**同步队列**，被移动的线程状态由 Waiting 变为 Blocked。
5. 从 wait 方法返回的前提是获得了调用对象的锁。

从上述细节可以看到，等待/通知机制依托于同步机制，其目的就是确保等待线程从 wait 方法返回后能够感知到通知线程对变量做出的修改。

该图描述了上面的步骤：

![](http://upload-images.jianshu.io/upload_images/4236553-c7c4441dc7ea979b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




WaitThread 获得了对象的锁，调用对象的 wait 方法，放弃了锁，进入的等待队列，然后 NotifyThread 拿到了对象的锁，然后调用对象的 notify 方法，将 WatiThread 移动到同步队列中，最后，NotifyThread 执行完毕，释放锁， WaitThread 再次获得锁并从 wait 方法返回继续执行。

到这里，关于应用层面的 wait 和 notify 基本就差不多了，后面的是关于虚拟机层面的抛砖引玉，涉及到 Java 的内置锁实现，synchronized 关键字底层实现，JVM 源码。算是本文的扩展吧。

注意：我们看到图中出现了 Monitor 这个词，也就是监视器，实际上，在 JDK 的注释中，也有 The current thread must own this object's monitor 这句话，当前线程必须拥有该对象的监视器。


如果我们编译这段含有 synchronized 关键字的代码，就会发现有一段代码被 monitorenter 指令和 monitorexit 指令括住了，这就是 synchronized 在编译期间做的事情，那么，在字节码被执行的时侯，该指令对应的 c 代码将会被执行。这里，我们必须打住，这里已经开始涉及到 synchronized 的相关原理了，本篇文章不会讨论这个。

wait noitfy 的答案都在 Java HotSpot 虚拟机的 C 代码中。但 R 大告诉我们不要轻易阅读虚拟机源码，众多细节可能会掩盖抽象，导致学习效率不高。如果同学们有兴趣，有大神写了3篇文章专门从 HotSpot 中解析源码，地址：

[Java的wait()、notify()学习三部曲之一：JVM源码分析](http://blog.csdn.net/boling_cavalry/article/details/77793224)，
 [Java的wait()、notify()学习三部曲之二：修改JVM源码看参数](http://blog.csdn.net/boling_cavalry/article/details/77897108)，
[Java的wait()、notify()学习三部曲之三：修改JVM源码控制抢锁顺序](http://blog.csdn.net/boling_cavalry/article/details/77995069)，
还有狼哥的 [JVM源码分析之Object.wait/notify实现](https://www.jianshu.com/p/f4454164c017).

上面四篇文章都从 JVM 的源码层面解析了 wait ，notify 的实现原理，非常清楚。

## 拾遗
1. wait(long) 方法，该方法参数是毫秒，也就是说，如果线程等待了指定的毫秒数，就会自动返回该线程。
2. wait（long, int）方法，该方法增加了纳秒级别的设置，算法是，前面的毫秒加上后面的纳秒，注意，是直接加一毫秒。
3. notify 方法调用后，如果等待的线程很多，JDK 源码中说将会随机找一个，但是 JVM 的源码中实际上是找第一个。
4. notifyAll 和 notify 不会立即生效，必须等到调用方执行完同步代码块，放弃锁之后才起作用。

## 总结

好了，关于 wait noitfy 的使用和基本原理就介绍到这里，不知道大家发现没有，并发和虚拟机高度相关。因此，可以说，学习并发的过程就是学习虚拟机的过程。而阅读虚拟机里的 openjdk 代码让人头大，但不管怎么样，丑媳妇迟早见公婆，openjdk 代码是一定要看的，加油！！！！



















































