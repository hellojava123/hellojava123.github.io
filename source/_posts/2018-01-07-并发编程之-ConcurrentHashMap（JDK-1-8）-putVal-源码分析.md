---
layout: post
title: 并发编程之-ConcurrentHashMap（JDK-1-8）-putVal-源码分析
date: 2018-01-07 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-399c7780d9dd810a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

我们之前分析了Hash的源码，主要是 put 方法。同时，我们知道，HashMap 在并发的时候是不安全的，为什么呢？因为当多个线程对 Map 进行扩容会导致链表成环。不单单是这个问题，当多个线程相同一个槽中插入数据，也是不安全的。而在这之后，我们学习了并发编程，而并发编程中有一个重要的东西，就是JDK 自带的并发容器，提供了线程安全的特性且比同步容器性能好出很多。一个典型的代表就是 ConcurrentHashMap，对，又是 HashMap ，但是这个 Map 是线程安全的，那么同样的，我们今天就看看该类的 put 方法是如何实现线程安全的。

## 源码加注释分析 putVal 方法

```java
 /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        // 死循环执行
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                // 初始化
                tab = initTable();
            // 获取对应下标节点，如果是kong，直接插入
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // CAS 进行插入
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果 hash 冲突了，且 hash 值为 -1，说明是 ForwardingNode 对象（这是一个占位符对象，保存了扩容后的容器）
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            // 如果 hash 冲突了，且 hash 值不为 -1
            else {
                V oldVal = null;
                // 同步 f 节点，防止增加链表的时候导致链表成环
                synchronized (f) {
                    // 如果对应的下标位置 的节点没有改变
                    if (tabAt(tab, i) == f) {
                        // 并且 f 节点的hash 值 不是大于0
                        if (fh >= 0) {
                            // 链表初始长度
                            binCount = 1;
                            // 死循环，直到将值添加到链表尾部，并计算链表的长度
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 如果 f 节点的 hasj 小于0 并且f 是 树类型
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // 链表长度大于等于8时，将该节点改成红黑树树
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        // 判断是否需要扩容
        addCount(1L, binCount);
        return null;
    }
```

楼主在代码中写了很多注释，但是还是说一下步骤（该方法和HashMap 的高度相似，但是多了很多同步操作）。

1. 校验key value 值，都不能是null。这点和 HashMap 不同。
2. 得到 key 的 hash 值。
3. 死循环并更新 tab 变量的值。
4.  如果容器没有初始化，则初始化。调用 initTable 方法。该方法通过一个变量 + CAS 来控制并发。稍后我们分析源码。
5. 根据 hash 值找到数组下标，如果对应的位置为空，就创建一个 Node 对象用CAS方式添加到容器。并跳出循环。
6. 如果 hash 冲突，也就是对应的位置不为 null，则判断该槽是否被扩容了（-1 表示被扩容了），如果被扩容了，返回新的数组。
7. 如果 hash 冲突 且 hash 值不是 -1，表示没有被扩容。则进行链表操作或者红黑树操作，注意，这里的 f 头节点被锁住了，保证了同时只有一个线程修改链表。防止出现链表成环。
8. 和 HashMap 一样，如果链表树超过8，则修改链表为红黑树。
9. 将数组加1（CAS方式），如果需要扩容，则调用 transfer 方法（非常复杂，以后再详解）进行移动和重新散列，该方法中，如果是槽中只有单个节点，则使用CAS直接插入，如果不是，则使用 synchronized 进行同步，防止并发成环。


这里说一说 initTable 方法：

```java
    /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 小于0说明被其他线程改了
            if ((sc = sizeCtl) < 0)
                // 自旋等待
                Thread.yield(); // lost initialization race; just spin
            // CAS 修改 sizeCtl 的值为-1
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        // sc 在初始化的时候用户可能会自定义，如果没有自定义，则是默认的
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        // 创建数组
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // sizeCtl 计算后作为扩容的阀值
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }


```

该方法为了在并发环境下的安全，加入了一个 sizeCtl 变量来进行判断，只有当一个线程通过CAS修改该变量成功后（默认为0，改成 -1），该线程才能初始化数组。保证了初始化数组时的安全性。

## 总结

ConcurrentHashMap 是并发大师 Doug Lea 的杰作，可以说鬼斧神工，总的来说，使用了 CAS 加 synchronized 来保证了 put 操作并发时的危险（特别是链表），相比 同步容器 hashTable 来说，如果容器大小是16，并发的性能是他的16倍，注意，读的时候是没有锁的，完全并发，而 HashTable 在 get 方法上直接加上了 synchronized 关键字，性能差距不言而喻。

当然，楼主这篇文章可能之写到了 ConcurrentHashMap 的皮毛，关于如何扩容，楼主没有详细介绍，而楼主在阅读源码的收获也很多，发现了很多有趣的东西，比如 ThreadLocalRandom 类在 addCount 方法中的应用，大家可以看看该类，非常的实用。

注意：这篇文章仅仅是 ConcurrentHashMap 的开头，关于 ConcurrentHashMap 里面的精华太多，值得我们好好学习。


good luck ！！！！！












