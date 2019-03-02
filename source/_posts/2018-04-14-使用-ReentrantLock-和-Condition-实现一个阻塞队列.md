---
layout: post
title: 使用-ReentrantLock-和-Condition-实现一个阻塞队列
date: 2018-04-14 11:11:11.000000000 +09:00
---
## 前言

从之前的阻塞队列的源码分析中，我们知道，JDK 中的阻塞队列是使用 ReentrantLock 和  Condition 实现了，我们今天来个简易版的。代码如下：

## 代码

```java
public class BoundedBuffer {

  final ReentrantLock lock = new ReentrantLock();
  final ConditionObject notFull = (ConditionObject) lock.newCondition();
  final ConditionObject notEmpty = (ConditionObject) lock.newCondition();

  final Object[] items = new Object[100];
  int putptr, takeptr, count;

  public void put(Object x) throws InterruptedException {
    lock.lock();
    try {
      // 当数组满了
      while (count == items.length) {
        // 释放锁，等待
        notFull.await();
      }
      // 放入数据
      items[putptr] = x;
      // 如果到最后一个位置了,下标从头开始,防止下标越界
      if (++putptr == items.length) {
        // 从头开始
        putptr = 0;
      }
      // 对 count ++ 加加
      ++count;
      // 通知 take 线程,可以取数据了,不必继续阻塞
      notEmpty.signal();
    } finally {
      lock.unlock();
    }
  }

  public Object take() throws InterruptedException {
    lock.lock();
    try {
      // 如果数组没有数据,则等待
      while (count == 0) {
        notEmpty.await();
      }
      // 取数据
      Object x = items[takeptr];
      // 如果到数组尽头了,就从头开始
      if (++takeptr == items.length) {
        // 从头开始
        takeptr = 0;
      }
      // 将数量减1
      --count;
      // 通知阻塞的 put 线程可以装填数据了
      notFull.signal();
      return x;
    } finally {
      lock.unlock();
    }
  }
```

**其实，这并不是我写的，而是 Condition 接口的 JavaDoc 文档中写的。并且文档中说，请不要再次实现这个队列，因为 JDK 内部已经是实现了。原话如下：**

>  (The {@link java.util.concurrent.ArrayBlockingQueue} class provides  this functionality, so there is no reason to implement this sample usage class.)

楼主只是觉得这个代码写的挺好的，所以分享一下，其中关键的还是 Condition 和重入锁，重入锁的核心源码我们已经看过了，今天来看看 Condition 的源码。所以我们今天使用了这个代码作为一个入口。

可以看到，Condition 的重要方法就是 await 和 signal，类似 Object 类的 wait 和 notify 方法。

我们后面会详细讲讲这两个方法的实现。




