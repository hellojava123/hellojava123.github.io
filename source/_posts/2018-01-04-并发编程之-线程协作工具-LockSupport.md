---
layout: post
title: 并发编程之-线程协作工具-LockSupport
date: 2018-01-04 11:11:11.000000000 +09:00
---
![LockSupport ](http://upload-images.jianshu.io/upload_images/4236553-db779066121b52b6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 前言

在前面的文章中，我们介绍了并发工具中的4个，Samephore，CyclicBarrier，CountDownLatch，Exchanger，但是我们漏了一个，非常的好用的工具，楼主在这里必须加上。

## LockSupport

LockSupport 是一个非常方便实用的线程**阻塞**工具，他可以在任意位置让线程阻塞。并且是静态的方法。是不是很心动？

LockSupport 的静态方法 park（）可以阻塞当前线程，类似的还有 parkNanos()，parkUntil(）等，他们实现了一个限时的等待。

同样的，有阻塞的方法，当然有唤醒的方法，什么呢？unpark（Thread） 方法。该方法可以将指定线程唤醒。

我们还是来一个例子吧，看看到底有多好用：

```java

public class LockSupportInterruptDemo {

  static Object u = new Object();
  static ChangeObjectThread t1 = new ChangeObjectThread("t1");
  static ChangeObjectThread t2 = new ChangeObjectThread("t2");


  static class ChangeObjectThread extends Thread {

    public ChangeObjectThread(String name) {
      super.setName(name);
    }

    public void run() {
      synchronized (u) {
        System.out.println("in " + getName());
        // wait
        LockSupport.park();
        if (Thread.interrupted()) {
          System.err.println(getName() + "被中断了");
        }
      }
      System.out.println(getName() + "执行结束了");
    }
  }

  public static void main(String[] args) throws InterruptedException {
    t1.start();
    Thread.sleep(1000);
    t2.start();
    Thread.sleep(3000);

    t1.interrupt();
    // notify
    LockSupport.unpark(t2);
  }

}

```

执行结果：

![](http://upload-images.jianshu.io/upload_images/4236553-c3437f99e0b59709.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

完全实现了 wait notify 的功能，但是，请注意，park 方法和 wait 方法相比，不需要获取某个对象的锁，也不会抛出 InterruptedException 异常，因此，你需要像我们的例子一样，使用静态方法进行判断。

如果你将 park 方法改成 park（this）/park（Thread），那么在打印 线程dump 信息的时候会打印阻塞对象的详细信息。

![](http://upload-images.jianshu.io/upload_images/4236553-ff56d22a18d12cf8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



该方法和 Lock 接口一样都是使用的 sun.misc.Unsafe 的 park 方法实现的阻塞。

还有一个需要注意的是：park 方法和 unpark 方法执行顺序不是那么的严格。比如我们在 Thread 类中提到的 suspend 方法 和resume 方法，如果顺序错误，将导致永远无法唤醒，但 park 方法和 unpark 方法则不会，我们测试一下，将 unpark 方法紧跟着 start 方法后面执行，那么也就是说，unpark 方法在 线程2 的park 方法之前执行，但结果相同。

![](http://upload-images.jianshu.io/upload_images/4236553-ea2b09a428891c44.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们将 unpark 方法移动到了 start 方法后面，依然正确执行。

什么原因呢？这是因为 LockSupport 使用了类似信号量的机制。他为每一个线程准备了一个许可（默认不可用），如果许可能用，那么 park 函数会立即返回，并且消费这个许可（也就是将许可**变为不可用**），如果许可不可用，将会阻塞。而 unpark 方法则使得一个许可**变为可用**（但是和信号量不同的是，许可不能累加，你不可能拥有超过要给许可，他永远只有一个）。

下面是JDK文档：

![park](http://upload-images.jianshu.io/upload_images/4236553-666a10cfddb4be6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![unpark](http://upload-images.jianshu.io/upload_images/4236553-4cdaf955622609ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个特定使得：即使 unpark 方法在 park 方法之前执行，他也可以使下一次的 park 操作立即返回。这也使上面的代码能正确执行的原因。


好了，到这里，LockSupport 就介绍完了，可以说，该方法可以替代 wait ，notify ，Condition 的 await ，signal 方法。注意，这里的 park 方法底层和 Lock 的底层实现是一致的。都是掉哟个 sun.misc.Unsafe。这个类可以说很牛逼。

good luck ！！！！ 














