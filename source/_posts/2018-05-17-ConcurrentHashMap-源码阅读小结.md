---
layout: post
title: ConcurrentHashMap-源码阅读小结
date: 2018-05-17 11:11:11.000000000 +09:00
---
## 前言

每一次总结都意味着重新开始，同时也是为了更好的开始。ConcurrentHashMap 一直是我心中的痛。虽然不敢说完全读懂了，但也看了几个重要的方法，有不少我觉得比较重要的知识点。

然后呢，放一些楼主写的关于 ConcurrentHashMap 相关源码分析的文章链接：
1. [ConcurrentHashMap 扩容分析拾遗](https://www.jianshu.com/p/dc69edc17b32)
1. [并发编程——ConcurrentHashMap#addCount() 分析](https://www.jianshu.com/p/749d1b8db066)
2. [并发编程——ConcurrentHashMap#transfer() 扩容逐行分析](https://www.jianshu.com/p/2829fe36a8dd)
3. [并发编程——ConcurrentHashMap#helpTransfer() 分析](https://www.jianshu.com/p/39b747c99d32)
4. [并发编程 —— ConcurrentHashMap size 方法原理分析](https://www.jianshu.com/p/88881fdfcf4c)
4. [并发编程之 ConcurrentHashMap（JDK 1.8） putVal 源码分析](https://www.jianshu.com/p/77fda250bddf)
5. [深入理解 HashMap put 方法（JDK 8逐行剖析）](https://www.jianshu.com/p/468c638e0e83)
6. [深入理解 hashcode 和 hash 算法](https://www.jianshu.com/p/eb9ab4211163)

## putVal 方法总结

说起 ConcurrentHashMap ，当然从入口开始说。该方法要点如下：

1. 不允许有 null key 和 null value。
2. 只有在第一次 put 的时候才初始化 table。初始化有并发控制。通过 sizeCtl 变量判断（小于 0）。
3. 当 hash 对应的下标是 null 时，使用  CAS 插入元素。 
4. 当 hash 对应的下标值是 forward 时，帮助扩容，但有可能帮不了，因为每个线程默认 16 个桶，如果只有 16个桶，第二个线程是无法帮助扩容的。
5. 如果 hash 冲突了，同步头节点，进行链表操作，如果链表长度达到 8 ，分成红黑树。
6. 调用 addCount 方法，对 size 加一，并判断是否需要扩容（如果是覆盖，就不调用该方法）。
7. Cmap 的并发性能是 hashTable 的 table.length 倍。只有出现链表才会同步，否则使用 CAS 插入。性能极高。

## size 方法总结

1. size 方法不准确，原因是由于并发插入，baseCount 难以及时更新。计数盒子也难以及时更新。
2. 内部通过两个变量，一个是 baseCount，一个是 counterCells，counterCells 是并发修改 baseCount 后的备用方案。
3. 具体更新 baseCount 和 counterCells 是在 addCount 方法中。备用方法 fullAddCount 则会死循环插入。
4. CounterCell 是一个用于分配计数的填充单元，改编自 LongAdder和Striped64。内部只有一个 volatile 的 value 变量，同时这个类标记了 `@sun.misc.Contended `，这是一个避免伪共享的注解，用于替代之前的缓存行填充。多线程情况下，注解让性能提升 5 倍。

## helpTransfer 方法总结

1. 当 Cmap 尝试插入的时候，发现该节点是 forward 类型，则会帮助其扩容。
2. 每次加入一个线程都会将 sizeCtl 的低 16 位加一。同时会校验高 16 位的标示符。
3. 扩容最大的帮助线程是  65535，这是低 16 位的最大值限制的。
4. 每个线程默认分配 16 个桶，如果桶的数量是 16，那么第二个线程无法帮助其扩容。

## transfer 方法总结

1. 该方法会根据 CPU 核心数平均分配给每个 CPU 相同数量的桶。但如果不够 16 个，默认就是 16 个。
2. 扩容是按照 2 倍进行扩容。
3. 每个线程在处理完自己领取的区间后，还可以继续领取，如果有的话。这个是 transferIndex 变量递减 16 实现的。
4. 每次处理空桶的时候，会插入一个 forward 节点，告诉 putVal 的线程：“我正在扩容，快来帮忙”。但如果只有 16 个桶，只能有一个线程扩容。
5. 如果有了占位符，那就不处理，跳过这个桶。
6. 如果有真正的实际值，那就同步头节点，防止 putVal 那里并发。
7. 同步块里会将链表拆成两份，根据 hash & length 得到是否是 0，如果是0，放在低位，反之，反之放在 length + i 的高位。这里的设计是为了防止下次取值的时候，hash 不到正确的位置。
8. 如果该桶的类型是红黑树，也会拆成 2 个，这是必须的。然后判断拆分过的桶的大小是否小于等于 6，如果是，改成链表。
9. 线程处理完之后，如果没有可选区间，且任务没有完成，就会将整个表检查一遍，防止遗漏。

## addCount 方法总结

1. 当插入结束的时候，会对 size 进行加一。也会进行是否需要扩容的判断。
2. 优先使用计数盒子（如果不是空，说明并发了），如果计数盒子是空，使用 baseCount 变量。对其加 X。
3. 如果修改 baseCount 失败，使用计数盒子。如果此次修改失败，在另一个方法死循环插入。
4. 检查是否需要扩容。
5. 如果 size 大于等于 sizeCtl 阈值，且长度小于 1 << 30，可以扩容成 1 << 30，但不能扩容成 1 << 31。
6. 如果已经在扩容，帮助其扩容，和 helpTransfer 逻辑一样。
7. 如果没有在扩容，自行开启扩容，更新 sizeCtl 变量为负数，赋值为标识符高 16 位 + 2。

## 小结

ConcurrentHashMap 满是财富，都是精华代码，我们这次阅读只是管中窥豹，要知道其中包含 53 个类，6300 行代码，但这次确实收获很多。有时间一定再次阅读！！ 

能力不高，水平有限，有些地方确实理解不了 Doug Lea 大师的设计，如果有什么错误，还请大家指出。不胜感激。






















