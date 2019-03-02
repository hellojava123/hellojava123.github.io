---
layout: post
title: 并发编程-——-ConcurrentHashMap-size-方法原理分析
date: 2018-04-16 11:11:11.000000000 +09:00
---
## 前言

ConcurrentHashMap 博大精深，从他的 50 多个内部类就能看出来，似乎 JDK 的并发精髓都在里面了。但他依然拥有体验良好的 API 给我们使用，程序员根本感觉不到他内部的复杂。但，他内部的每一个方法都复杂无比，就连 size 方法，都挺复杂的。

今天就一起来看看这个 size 方法。

## size 方法

代码如下：

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 : (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int)n);
}
```

最大返回 int 最大值，但是这个 Map 的长度是有可能超过 int 最大值的，所以 JDK 8 增了 mappingCount 方法。代码如下：

```java
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}
```

相比较 size 方法，mappingCount 方法的返回值是 long 类型。所以不必限制最大值必须是 Integer.MAX_VALUE。而 JDK 推荐使用这个方法。但这个返回值依然不一定绝对准确。

从这两个方法中可以看出，sumCount 方法是核心。

## sumCount 方法实现

代码如下：

```java
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

上面的方法逻辑：当 counterCells 不是 null，就遍历元素，并和 baseCount 累加。

两个属性 ： baseCount 和 counterCells。

先看 baseCount。

```java
    /**
     * Base counter value, used mainly when there is no contention,
     * but also as a fallback during table initialization
     * races. Updated via CAS.
     * 当没有争用时，使用这个变量计数。
     */
    private transient volatile long baseCount;
```

一个 volatile 的变量，在 addCount 方法中会使用它，而 addCount 方法在 put 结束后会调用。在 addCount 方法中，会对这个变量做 CAS 加法。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-a40dedc4bbf3307f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



但是如果并发导致 CAS 失败了，怎么办呢？使用 counterCells。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-ac8855d5dcf40996.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


如果上面 CAS 失败了，在 fullAddCount 方法中，会继续死循环操作，直到成功。

![image.png](https://upload-images.jianshu.io/upload_images/4236553-a9a7cd9076a24321.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


而这个 CounterCell 类又是上面鬼呢？

```java
// 一种用于分配计数的填充单元。改编自LongAdder和Striped64。请查看他们的内部文档进行解释。
@sun.misc.Contended 
static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

使用了 @sun.misc.Contended  标记的类，内部一个 volatile 变量。注释说，改编自LongAdder和Striped64,关于这两个类，请看 [Java8 Striped64 和 LongAdder](http://ifeve.com/java8-striped64-and-longadder/)。

而关于这个注解，有必要解释一下。这个注解标识着这个类防止需要防止 "伪共享".

说说伪共享。引用 一下别人的说法：

> 避免伪共享(false sharing)。
先引用个伪共享的解释：
缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，
一般为32-256个字节。最常见的缓存行大小是64个字节。当多线程修改互相独立的变量时，
如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

所以伪共享对性能危害极大。

JDK 8 版本之前没有这个注解，Doug Lea 使用拼接来解决这个问题，把缓存行加满，让缓存之间的修改互不影响。

在我的机器上测试，加和不加这个注解的性能差距达到了 5 倍。


## 总结

好了，关于 Size 方法就简单介绍到这里。总结一下：

JDK 8 推荐使用mappingCount 方法，因为这个方法的返回值是 long 类型，不会因为 size 方法是 int 类型限制最大值（size 方法是接口定义的，不能修改）。

在没有并发的情况下，使用一个 baseCount volatile 变量就足够了，当并发的时候，CAS 修改 baseCount 失败后，就会使用 CounterCell 类了，会创建一个这个对象，通常对象的 volatile value 属性是 1。在计算 size 的时候，会将 baseCount 和 CounterCell 数组中的元素的 value 累加，得到总的大小，但这个数字仍旧可能是不准确的。

还有一个需要注意的地方就是，这个 CounterCell 类使用了 @sun.misc.Contended  注解标识，这个注解是防止伪共享的。是 1.8 新增的。使用时，需要加上 ` -XX:-RestrictContended` 参数。




















