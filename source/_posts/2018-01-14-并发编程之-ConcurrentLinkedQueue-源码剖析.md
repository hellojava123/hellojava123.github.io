---
layout: post
title: 并发编程之-ConcurrentLinkedQueue-源码剖析
date: 2018-01-14 11:11:11.000000000 +09:00
---
## 前言

今天我们继续分析 java 并发包的源码，今天的主角是谁呢？ConcurrentLinkedQueue，上次我们分析了并发下 ArrayList 的替代 CopyOnWriteArrayList，这次分析则是并发下 LinkedArrayList 的替代 ConcurrentLinkedQueue， 也就是并发链表。

## Demo

![Demo](http://upload-images.jianshu.io/upload_images/4236553-f135956c72297b69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类继承结构如下：

![继承图](http://upload-images.jianshu.io/upload_images/4236553-af008a310444c570.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类是 Collection 框架下的实现。也就是Java 类库提供的数据结构。

add 方法将指定元素插入此队列的尾部。
poll 方法 获取并移除此队列的头，如果此队列为空，则返回 null。
peek 方法  获取但不移除此队列的头；如果此队列为空，则返回 null。

那么我们就看看 doug lea 是如何实现并发安全的吧。在这之前，我们可以试想一下，实现并发安全无非两种方式，一种是锁，就像我们之前分析的容器，比如 concurrentHashMap，CopyOnWriteArrayList ， LinkedBolckingQueue，还有一种是 CAS，在这些容器里也用到了。那么，如果是我们来实现这个队列，使用什么方式呢？有趣的问题。

开始看源码吧。

## add 方法源码剖析

实际上是调用 offer 方法，add 方法是 Collection 接口规定的容器方法，而 offer 方法是 Queue 接口的方法。

![add方法](http://upload-images.jianshu.io/upload_images/4236553-ae672d951d966e6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那我们就看看 offer 方法：

```java
    public boolean offer(E e) {
        // 检查是否是null，如果是null ，抛出NullPointerException
        checkNotNull(e);
        // 创建一个node 对象，使用  CAS 创建对象
        final Node<E> newNode = new Node<E>(e);
        // 轮询链表节点，知道找到节点的 next 为null，才会进行赋值
        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {
                // 找到null值之后将刚刚创建的值通过CAS放入
                if (p.casNext(null, newNode)) {
                    // 因为 p 遍历在轮询后会变化，因此需要判断，如果不相等，则使用CAS将新节点作为尾部节点。
                    if (p != t)
                        casTail(t, newNode);  // Failure is OK.
                     // 放入成功后返回 ture
                    return true;
                }
            }
            // 轮询后  p 有可能等于 q，此时，就需要对 p 重新赋值。
            else if (p == q)
                // 这里需要注意一下：判断t != t，是因为并发下可能 tail 被改了，如果被改了，则使用新的 t，否则从链表头重新轮询。
                p = (t != (t = tail)) ? t : head;
            else
                // 同样，当 t 不等于 p 时，说明 p 在上面被重新赋值了，并且 tail 也被别的线程改了，则使用新的 tail，否则循环检查p的下个节点
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```

代码行数很少，楼主注释也写了，这里可以看到 doug lea 使用了 CAS 的方式防止并发错误，同时，也看得出对 tail 变量被修改的担忧，通过 t != t 的判断，来检查 tail 是否被其他线程修改了，而这个offer 操作，如果不成功，则永远不会返回，这个队列同时也是无界的。这点在使用的时候需要注意一下。

那么 poll 方法如何实现呢？


## poll 方法源码剖析

```java
    public E poll() {
        // 循环跳出标记，类似goto
        restartFromHead:
        // 死循环
        for (;;) {
            // 死循环，从 head 开始遍历
            for (Node<E> h = head, p = h, q;;) {
                E item = p.item;
                // 如果 head 不是null 且 将 head 的 item 属性设置为null成功，则返回并更新头节点
                if (item != null && p.casItem(item, null)) {
                    // 如果 p ！= h 说明在 p 轮询时被修改了
                    if (p != h) 
                         // 如果p 的next 属性不是null ，将 p 作为头节点，而 q 将会消失
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                // 如果 p（head） 的 next 节点 q 也是null，则表示没有数据了，返回null，则将 head 设置为null
                // 注意：  updateHead 方法最后还会将原有的 head 作为自己 next 节点，方便offer 连接。
                else if ((q = p.next) == null) {
                    updateHead(h, p);
                    return null;
                }
                // 如果 p == q，说明别的线程取出了 head，并将 head 更新了。就需要重新开始
                else if (p == q)
                    // 从头开始重新循环
                    continue restartFromHead;
               // 如果都不是，则将 h 的 next 赋给 h，并重新循环。
                else
                    p = q;
            }
        }
    }
```

上面楼主已经写了注释，但是有一个非常困扰哦楼主的疑点，就是 else if （p == q） 这行代码，楼主分析的没有问题，但是再楼主的单线程测试这段代码时，出现了诡异的情况，无法解释，因此， 楼主贴出测试用例，大家一起看看：

测试代码：

![](http://upload-images.jianshu.io/upload_images/4236553-273a503d07bd092f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

断点代码：

![](http://upload-images.jianshu.io/upload_images/4236553-6003b2d5ea788117.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意，断点位置一定要和我的一致。会出现一些奇怪的效果。楼主无法解释，因为这个问题，楼主一直都不敢发这篇文章出来，但楼主觉得有必要说出这个问题，抛砖引玉。

问题在于：单线程怎么会进入这段代码？按道理，但线程是不会出现这个情况的。

## 总结

这次的源码分析让楼主很痛苦，网上很多的文章也无法解释这是为什么，希望有高人能告诉楼主，到底是怎么回事？

































