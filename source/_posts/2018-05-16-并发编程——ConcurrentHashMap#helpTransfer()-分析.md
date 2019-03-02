---
layout: post
title: 并发编程——ConcurrentHashMap#helpTransfer()-分析
date: 2018-05-16 11:11:11.000000000 +09:00
---
## 前言

ConcurrentHashMap 鬼斧神工，并发添加元素时，如果 map 正在扩容，其他线程甚至于还会帮助扩容，也就是多线程扩容。就这一点，就可以写一篇文章好好讲讲。今天一起来看看。

## 源码分析

为什么帮助扩容？

在 putVal 方法中，如果发现线程当前 hash 冲突了，也就是当前 hash 值对应的槽位有值了，且如果这个值是 -1 （MOVED），说明 Map 正在扩容。那么就帮助 Map 进行扩容。以加快速度。

具体代码如下：

```java
int hash = spread(key.hashCode());
int binCount = 0;
for (Node<K,V>[] tab = table;;) {
    Node<K,V> f; int n, i, fh;
    if (tab == null || (n = tab.length) == 0)
        tab = initTable();
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
        if (casTabAt(tab, i, null,
                     new Node<K,V>(hash, key, value, null)))
            break;                   // no lock when adding to empty bin
    } // 如果对应槽位不为空，且他的 hash 值是 -1，说明正在扩容，那么就帮助其扩容。以加快速度
    else if ((fh = f.hash) == MOVED)
        tab = helpTransfer(tab, f);
```

传入的参数是：成员变量 table 和 对应槽位的 f 变量。

怎么验证 hash 值是 MOVED 就是正在扩容呢?

在 Cmap（ConcurrentHashMap 简称） 中，定义了一堆常量，其中：

```java
static final int MOVED     = -1; // hash for forwarding nodes
```

`hash for forwarding nodes`，说明这个为了移动节点而准备的常量。

在 Node 的子类 ForwardingNode 的构造方法中，可以看到这个变量作为 hash 值进行了初始化。

```java
ForwardingNode(Node<K,V>[] tab) {
    super(MOVED, null, null, null);
    this.nextTable = tab;
}
```
 
而这个构造方法只在一个地方调用了，即 transfer（扩容） 方法。

点到为止。

关于扩容后面再开一篇。

好了，如何帮助扩容呢？那要看看 `helpTransfer ` 方法的实现。

```java
/**
 * Helps transfer if a resize is in progress.
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 如果 table 不是空 且 node 节点是转移类型，数据检验
    // 且 node 节点的 nextTable（新 table） 不是空，同样也是数据校验
    // 尝试帮助扩容
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
    	// 根据 length 得到一个标识符号
        int rs = resizeStamp(tab.length);
    	// 如果 nextTab 没有被并发修改 且 tab 也没有被并发修改
    	// 且 sizeCtl  < 0 （说明还在扩容）
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
        	// 如果 sizeCtl 无符号右移  16 不等于 rs （ sc前 16 位如果不等于标识符，则标识符变化了）
        	// 或者 sizeCtl == rs + 1  （扩容结束了，不再有线程进行扩容）（默认第一个线程设置 sc ==rs 左移 16 位 + 2，当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1）
        	// 或者 sizeCtl == rs + 65535  （如果达到最大帮助线程的数量，即 65535）
        	// 或者转移下标正在调整 （扩容结束）
        	// 结束循环，返回 table
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 如果以上都不是, 将 sizeCtl + 1, （表示增加了一个线程帮助其扩容）
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
            	// 进行转移
                transfer(tab, nextTab);
                // 结束循环
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

关于 sizeCtl 变量：

-1 :代表table正在初始化,其他线程应该交出CPU时间片
-N: 表示正有N-1个线程执行扩容操作（高 16 位是 length 生成的标识符，低 16 位是扩容的线程数）
大于 0: 如果table已经初始化,代表table容量,默认为table大小的0.75,如果还未初始化,代表需要初始化的大小


代码步骤：
1. 判 tab 空，判断是否是转移节点。判断 nextTable 是否更改了。
2. 更加 length 得到标识符。
3. 判断是否并发修改了，判断是否还在扩容。
4. 如果还在扩容，判断标识符是否变化，判断扩容是否结束，判断是否达到最大线程数，判断扩容转移下标是否在调整（扩容结束），如果满足任意条件，结束循环。
4. 如果不满足，并发转移。


这里有一个花费了很长时间纠结的地方：

```java
sc == rs + 1
```

这个判断可以在 addCount 方法中找到答案：默认第一个线程设置 sc ==rs 左移 16 位 + 2，当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1。

如果 sizeCtl  == 标识符 + 1 ，说明库容结束了，没有必要再扩容了。


总结一下：

当 Cmap put 元素的时候，如果发现这个节点的元素类型是 forward 的话，就帮助正在扩容的线程一起扩容，提高速度。其中， sizeCtl 是关键，该变量高 16 位保存 length 生成的标识符，低 16 位保存并发扩容的线程数，通过这连个数字，可以判断出，是否结束扩容了。

如下图：

![](https://upload-images.jianshu.io/upload_images/4236553-9fd0441a1f41116b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



