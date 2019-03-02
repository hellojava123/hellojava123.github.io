---
layout: post
title: 深入理解-HashMap-put-方法（JDK-8逐行剖析）
date: 2017-12-31 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-208976630b64d450.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## **前言**

注意：我们今天所有的一切都是基于 JDK 8，JDK 8 的实现和 JDK 7 有重大区别。

前面我们分析了 hashCode 和 hash 算法的原理，其实都是为我们解析 HashMap 做铺垫，因为 HashMap 确实比较复杂（如果你每一行代码都看的话，每个位移都纠结的话），虽然总的来说，HashMap 不过是 Node 数组加 链表和红黑树。但是里面的细节确是无比的优雅和有趣。楼主为什么选择 put 方法来讲呢？因为从楼主看来，HashMap 的精髓就在 put 方法中。

HashMap 的解析从楼主来看，主要可以分为几个部分：
1. hash 算法（这个我们之前说过了，今天就不再赘述了）
2. 初始化数组。
3. 通过 hash 计算下标并检查 hash 是否冲突，也就是对应的下标是否已存在元素。
4. 通过判断是否含有元素，决定是否创建还是追加链表或树。
5. 判断已有元素的类型，决定是追加树还是追加链表。
6. 判断是否超过阀值，如果超过，则重新散列数组。
7. Java 8 重新散列时是如何优化的。


开始吧！！！

## **1. 初始化数组**

首先我们来一个测试例子：

```java
  public static void main(String[] args) {
    HashMap<String, Integer> hashMap = new HashMap<>(2);
    hashMap.put("one", 1);
    Integer one = hashMap.get("one");
    System.out.println(one);
  }
}

```

一个简单的不能再简单的使用 HashMap 的例子，其中包含了对于 HashMap 来说关键的 3 个步骤，初始化，put 元素，get 元素。

由于我们预计会放入一个元素，出于性能考虑，我们将容量设置为 2，既保证了性能，也节约了空间（置于为什么，我们在之前的文章中讲过）。

那么我们就看看 new 操作的时候做了些什么事情：

```java
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity.
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

//=================================================================================
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```

上面是 HashMap 的两个构造方法，其中，我们设置了初始容量为 2， 而默认的加载因子我们之前说过：0.75，当然也可以自己设置，但 0.75 是最均衡的设置，没有特殊要求不要修改该值，加载因子过小，理论上能减少 hash 冲突，加载因子过大可以节约空间，减少 HashMap 中最耗性能的操作：reHash。

从代码中我可以看到，如果我们设置的初始化容量小于0，将会抛出异常，如果加载因子小于0也会抛出异常。同时，如果初始容量大于最大容量，则重新设置为最大容量。

我们开最后两行代码，首先，对负载因子进行赋值，这个没什么可说的。
牛逼的是下面一行代码：this.threshold = tableSizeFor(initialCapacity); 可以看的出来这个动作是计算阀值，上面是阀值呢？阀值就是，如果容器中的元素大于阀值了，就需要进行扩容，那么这里的这行代码，就是根据初始容量进行阀值的计算。

我们进入到该方法查看：

```java
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

一通或运算和无符号右移运算，那么这个运算的的最后结果是什么呢？这里其实就是如果用户输入的值不是2的幂次方（我们通过之前的分析，应该直到初始容量如果不是2的幂次方会有多么不好的结果）。通过位移运算和或运算，最后得到一定是2的幂次方，并且是那个离那个数最近的数字，我们仔细看看该方法：

> | : 或运算，第一个操作数的的第n位于第二个操作数的第n位 只要有一个是1，那么结果的第n为也为1，否则为0

首先，将容量减1，我们后面就知道了。然后将该数字无符号右移1，2，4，8，16，总共移了32位，刚好是一个int类型的长度。在移动的过程中，还对该数字进行或运算，为了方便查看，楼主写一下2进制的运算过程，假如我输入的是10，明显不是2的幂次方。我们看看会怎么样：

10 = 1010；
n = 9;

1001 == 9；

1001 >>> 1 = 0100;
1001 或 0100 = 1101；

1101 >>> 2 = 0011;
110 或 0011 = 1111；

1111 >>> 4 = 0000;
1111 或 0000 = 1111；

1111 >>> 8 = 0000;
1111 或 0000 = 1111；

1111 >>> 16 = 0000；
1111 或 0000 = 1111；

最后，1111 也就是 15 ，15 + 1 = 16，刚好就是距离10 最近的并且没有变小的2的幂次方数。可以说这个算法非常的牛逼。楼主五体投地。

但是如果是 16 呢，并且没有不减一，我们看看什么结果：

16 = 10000； 


10000 >>> 1 = 01000; 
10000 或 01000 = 11000；

11000 >>> 2 = 00110; 
11000 或 00110 = 11110；

11110 >>> 4 = 00001; 
11110 或 00001 = 11111；

11111 >>> 8 = 00000; 
11111 或 00000 = 11111；

11111 >>> 16 = 00000； 
11111 或 00000 = 11111；

最后的数字就是31 ，31+ 1 = 32，同样也是上升到了更大的2次幂的数字。但是这不是我想要的结果，所以，JDK 的作者在之前先减去了1. 防止出现这样的问题。
 
我们仔细观察其算法的过程，可以说，任何一个int  数字，都能找到离他最近的 2 的幂次方数字（并且比他大）。

好了。到这里就完成了初始化，不过请注意，这里设置的阀值并不是最终的阀值，最终的阀值我们会在后面详细说明。这里我们更加关注这个算法。真的牛逼啊。


## **2. 通过 hash 计算下标并检查 hash 是否冲突，也就是对应的下标是否已存在元素。**

初始化好了 HashMap，我们接着就调用了 put 方法，该方法如下：

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

```

其中调用 hash 方法，该方法我们之前已经深入讨论过，今天就不赘述了，如果有同学没有看过，也不要紧，看完这篇 再去看 或者 看完那篇 再来 看这篇都可以。有点绕。好，然后调用了 puVal 方法，我们看看该方法：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 当前对象的数组是null 或者数组长度时0时，则需要初始化数组
        if ((tab = table) == null || (n = tab.length) == 0)
            // 得到数组的长度 16
            n = (tab = resize()).length;
        // 如果通过hash值计算出的下标的地方没有元素，则根据给定的key 和 value 创建一个元素
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else { // 如果hash冲突了
            Node<K,V> e; K k;
            // 如果给定的hash和冲突下标中的 hash 值相等并且 （已有的key和给定的key相等（地址相同，或者equals相同）），说明该key和已有的key相同
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                // 那么就将已存在的值赋给上面定义的e变量
                e = p;
            // 如果以存在的值是个树类型的，则将给定的键值对和该值关联。
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // 如果key不相同，只是hash冲突，并且不是树，则是链表
            else { 
                // 循环，直到链表中的某个节点为null，或者某个节点hash值和给定的hash值一致且key也相同，则停止循环。
                for (int binCount = 0; ; ++binCount) {
                    // 如果next属性是空
                    if ((e = p.next) == null) {
                        // 那么创建新的节点赋值给已有的next 属性
                        p.next = newNode(hash, key, value, null);
                        // 如果树的阀值大于等于7，也就是，链表长度达到了8（从0开始）。
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 如果链表长度达到了8，且数组长度小于64，那么就重新散列，如果大于64，则创建红黑树
                            treeifyBin(tab, hash);
                        // 结束循环
                        break;
                    }
                    // 如果hash值和next的hash值相同且（key也相同）
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        // 结束循环
                        break;
                    // 如果给定的hash值不同或者key不同。
                    // 将next 值赋给 p，为下次循环做铺垫
                    p = e;
                }
            }
            // 通过上面的逻辑，如果e不是null，表示：该元素存在了(也就是他们呢key相等)
            if (e != null) { // existing mapping for key
                // 取出该元素的值
                V oldValue = e.value;
                // 如果 onlyIfAbsent 是 true，就不要改变已有的值，这里我们是false。
                // 如果是false，或者 value 是null
                if (!onlyIfAbsent || oldValue == null)
                    // 将新的值替换老的值
                    e.value = value;
                // HashMap 中什么都不做
                afterNodeAccess(e);
                // 返回之前的旧值
                return oldValue;
            }
        }
        // 如果e== null，需要增加 modeCount 变量，为迭代器服务。
        ++modCount;
        // 如果数组长度大于了阀值
        if (++size > threshold)
            // 重新散列
            resize();
        // HashMap 中什么都不做
        afterNodeInsertion(evict);
        // 返回null
        return null;
    }
```

该方法可以说是 HashMap 的核心方法，楼主已经在该方法中写满了注释。楼主说一下该方法的步骤：

1. 判断数组是否为空，如果是空，则创建默认长度位 16 的数组。
2. 通过与运算计算对应 hash 值的下标，如果对应下标的位置没有元素，则直接创建一个。
3. 如果有元素，说明 hash 冲突了，则再次进行 3 种判断。
	1. 判断两个冲突的key是否相等，equals 方法的价值在这里体现了。如果相等，则将已经存在的值赋给变量e。最后更新e的value，也就是替换操作。
	2. 如果key不相等，则判断是否是红黑树类型，如果是红黑树，则交给红黑树追加此元素。
	3. 如果key既不相等，也不是红黑树，则是链表，那么就遍历链表中的每一个key和给定的key是否相等。如果，链表的长度大于等于8了，则将链表改为红黑树，这是Java8 的一个新的优化。
4. 最后，如果这三个判断返回的 e 不为null，则说明key重复，则更新key对应的value的值。
5. 对维护着迭代器的modCount 变量加一。
6. 最后判断，如果当前数组的长度已经大于阀值了。则重新hash。



## **3. 通过判断是否含有元素，决定是否创建还是追加链表或树。**

首先判断是否含有元素，通过什么判断呢？ 

tab[i = (n - 1) & hash]；

这个算式根据 hash 值获取对应的下标，具体是什么原理，我们在上一篇文章已经说了原因。这里也不在赘述了。如果 hash 值没有冲突，则创建一个 Node 对象，参数是hash值，key，value，还有为null 的next 属性。下面是构造函数。

```java
	// 构造函数
     Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
```
如果没有冲突，后面紧接着就走 ++modCount，然后，判断容量是否大于阀值（默认是12）。如果大于，则调用 resize 方法，重新散列。resize 方法我们后面详细分析。










## **4. 判断已有元素的类型，决定是追加树还是追加链表。**

如果 hash 冲突了怎么办？我们刚刚说会有3种判断：

1. 判断两个冲突的key是否相等，equals 方法的价值在这里体现了。如果相等，则将已经存在的值赋给变量e。最后更新e的value，也就是替换操作。
2. 如果key不相等，则判断是否是红黑树类型，如果是红黑树，则交给红黑树追加此元素。
3. 如果key既不相等，也不是红黑树，则是链表，那么就遍历链表中的每一个key和给定的key是否相等。如果，链表的长度大于等于8了，则将链表改为红黑树，这是Java8 的一个新的优化。

注意：在链表的循环中，有一个方法 treeifyBin，这个方法在链表长度大于等于8 的时候会调用，那么该方法的内容是什么呢？

```java
 final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // 如果数组是null 或者数组的长度小于 64
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            // 重新散列
            resize();
        // 如果给定的hash冲突了，则创建红黑树结构
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }

```

该方法虽然主要功能是替换链表结构为红黑树，但是在替换前，会先判断，如果数组是 null 或者数组的长度小于 64，则重新散列，因为重新散列会拆分链表，使得链表的长度变短。提高性能。如果长度大于64了。就只能将链表变为红黑树了。





## **5. 判断是否超过阀值，如果超过，则重新散列数组。**

最后，判断是否阀值，如果超过则进行散列；

```java
        // 如果 e == null，需要增加 modeCount 变量，为迭代器服务。
        ++modCount;
        // 如果数组长度大于了阀值
        if (++size > threshold)
            // 重新散列
            resize();
        // HashMap 中什么都不做
        afterNodeInsertion(evict);
        // 返回null
        return null;

```

我们知道，阀值默认是16，那么 resize 方法就是重新散列的核心方法，我们看看该方法实现：

```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        // 如果老的容量大于0
        if (oldCap > 0) {
            // 如果容量大于容器最大值
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 阀值设为int最大值
                threshold = Integer.MAX_VALUE;
                // 返回老的数组，不再扩充
                return oldTab;
            }// 如果老的容量*2 小于最大容量并且老的容量大于等于默认容量
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 新的阀值也再老的阀值基础上*2
                newThr = oldThr << 1; // double threshold
        }// 如果老的阀值大于0
        else if (oldThr > 0) // initial capacity was placed in threshold
            // 新容量等于老阀值
            newCap = oldThr;
        else {  // 如果容量是0，阀值也是0，认为这是一个新的数组，使用默认的容量16和默认的阀值12           
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 如果新的阀值是0，重新计算阀值
        if (newThr == 0) {
            // 使用新的容量 * 负载因子（0.75）
            float ft = (float)newCap * loadFactor;
            // 如果新的容量小于最大容量 且 阀值小于最大 则新阀值等于刚刚计算的阀值，否则新阀值为 int 最大值
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        } 
        // 将新阀值赋值给当前对象的阀值。
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            // 创建一个Node 数组，容量是新数组的容量（新容量要么是老的容量，要么是老容量*2，要么是16）
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        // 将新数组赋值给当前对象的数组属性
        table = newTab;
        // 如果老的数组不是null
        if (oldTab != null) {
          // 循环老数组
            for (int j = 0; j < oldCap; ++j) {
                // 定义一个节点
                Node<K,V> e;
                // 如果老数组对应下标的值不为空
                if ((e = oldTab[j]) != null) {
                    // 设置为空
                    oldTab[j] = null;
                    // 如果老数组没有链表
                    if (e.next == null)
                        // 将该值散列到新数组中
                        newTab[e.hash & (newCap - 1)] = e;
                    // 如果该节点是树
                    else if (e instanceof TreeNode)
                        // 调用红黑树 的split 方法，传入当前对象，新数组，当前下标，老数组的容量，目的是将树的数据重新散列到数组中
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // 如果既不是树，next 节点也不为空，则是链表，注意，这里将优化链表重新散列（java 8 的改进）
                      // Java8 之前，这里曾是并发操作会出现环状链表的情况，但是Java8 优化了算法。此bug不再出现，但并发时仍然不建议HashMap
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 这里的判断需要引出一些东西：oldCap 假如是16，那么二进制为 10000，扩容变成 100000，也就是32.
                            // 当旧的hash值 与运算 10000，结果是0的话，那么hash值的右起第五位肯定也是0，那么该于元素的下标位置也就不变。
                            if ((e.hash & oldCap) == 0) {
                                // 第一次进来时给链头赋值
                                if (loTail == null)
                                    loHead = e;
                                else
                                    // 给链尾赋值
                                    loTail.next = e;
                                // 重置该变量
                                loTail = e;
                            }
                            // 如果不是0，那么就是1，也就是说，如果原始容量是16，那么该元素新的下标就是：原下标 + 16（10000b）
                            else {
                                // 同上
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 理想情况下，可将原有的链表拆成2组，提高查询性能。
                        if (loTail != null) {
                            // 销毁实例，等待GC回收
                            loTail.next = null;
                            // 置入bucket中
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

```

该方法可以说还是比较复杂的。初始的时候也是调用的这个方法，当链表数超过8的时候同时数组长度小于64的时候也是调用的这个方法。该方法步骤如下：

1. 判断容量是否大于0，如果大于0，并且容量已将大于最大值，则设置阀值为 int 最大值，并返回，如果老的容量乘以 2 小于最大容量，且老的容量大于等于16，则更新阀值。也就是乘以2.
2. 如果老的阀值大于0，则新的容量等于老的阀值。注意：这里很重要。还记的我们之前使用new 操作符的时候，会设置阀值为 2 的幂次方，那么这里就用上了那个计算出来的数字，也就是说，就算我们设置的不是2的幂次方，HashMap 也会自动将你的容量设置为2的幂次方。
3. 如果老的阀值和容量都不大于0，则认为是一个新的数组，默认初始容量为16，阀值为 16 * 0.75f，也就是 12。
4. 如果，新的阀值还是0，那么就使用我们刚刚设置的容量（HashMap 帮我们算的），通过乘以 0.75，得到一个阀值，然后判断算出的阀值是否合法：如果容量小于最大容量并且阀值小于最大容量，那么则使用该阀值，否则使用 int 最大值。
5. 将刚刚的阀值设置打当前Map实例的阀值属性中。
6. 将刚刚的数组设置到当前Map实例的数组属性中。
7. 如果老的数组不是null，则将老数组中的值重新散列到新数组中。如果是null，直接返回新数组。

那么，将老鼠组重新散列的过程到底是怎么样的呢？

## **6. Java 8 重新散列时是如何优化的。**

重新散列的代码：

```java
        // 如果老的数组不是null
        if (oldTab != null) {
          // 循环老数组
            for (int j = 0; j < oldCap; ++j) {
                // 定义一个节点
                Node<K,V> e;
                // 如果老数组对应下标的值不为空
                if ((e = oldTab[j]) != null) {
                    // 设置为空
                    oldTab[j] = null;
                    // 如果老数组没有链表
                    if (e.next == null)
                        // 将该值散列到新数组中
                        newTab[e.hash & (newCap - 1)] = e;
                    // 如果该节点是树
                    else if (e instanceof TreeNode)
                        // 调用红黑树 的split 方法，传入当前对象，新数组，当前下标，老数组的容量，目的是将树的数据重新散列到数组中
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // 如果既不是树，next 节点也不为空，则是链表，注意，这里将优化链表重新散列（java 8 的改进）
                      // Java8 之前，这里曾是并发操作会出现环状链表的情况，但是Java8 优化了算法。此bug不再出现，但并发时仍然不建议HashMap
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 这里的判断需要引出一些东西：oldCap 假如是16，那么二进制为 10000，扩容变成 100000，也就是32.
                            // 当旧的hash值 与运算 10000，结果是0的话，那么hash值的右起第五位肯定也是0，那么该于元素的下标位置也就不变。
                            if ((e.hash & oldCap) == 0) {
                                // 第一次进来时给链头赋值
                                if (loTail == null)
                                    loHead = e;
                                else
                                    // 给链尾赋值
                                    loTail.next = e;
                                // 重置该变量
                                loTail = e;
                            }
                            // 如果不是0，那么就是1，也就是说，如果原始容量是16，那么该元素新的下标就是：原下标 + 16（10000b）
                            else {
                                // 同上
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 理想情况下，可将原有的链表拆成2组，提高查询性能。
                        if (loTail != null) {
                            // 销毁实例，等待GC回收
                            loTail.next = null;
                            // 置入bucket中
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
```

这里楼主重新贴了上面的代码，因为这段代码比较重要。还是说一下该部分代码的步骤。

1. 首先循环老数组。下标从0开始，如果对应下标的值不是null，则判断该值有没有next 节点，也就是判断是否是链表。
2. 如果不是链表，则根据新的数组长度重新hash该元素。
3. 如果该节点是树，则调用红黑树的 split 方法，传入当前对象和新数组还有下标，该方法会重新计算红黑树中的每个hash值，如果重新计算后，树中元素的hash值不同，则重新散列到不同的下标中。达到拆分红黑树的目的，提高性能。具体如何拆分下面再说。
4. 之后的判断就是链表，在Java8中，该部分代码不是简单的将旧链表中的数据拷贝到新数组中的链表就完了，而是会对旧的链表进行重新 hash，如果 hash 得到的值和之前不同，则会从旧的链表中拆出，放到另一个下标中去，提高性能，刚刚的红黑树也是这么做的。

这里的重新hash 不是使用的 [e.hash & (newCap - 1)]  方法，而是使用更加效率的方法，直接 hash 老的数组容量，就没有了减一的操作，可见 JDK 的作者为了性能可以说是无所不用其极了。

其实我们的注释已经写了，但是楼主还是再解释一遍吧：

仔细阅读下面这段话：

oldCap 假如是16，那么二进制为 10000，扩容变成 100000，也就是32.当旧的hash值 与运算 10000，结果是0的话，那么hash值的右起第五位肯定也是0，那么该于元素的下标位置也就不变。但如果不是0是1的话，说明该hash值变化了，那么就需要对这个元素重新散列放置。那么应该放哪里呢？如果是16，那么最左边是1的话，说明hash值变大了16，那么只需要在原有的基础上加上16便好了。



这段代码还有一个需要注意的地方：在JDK 7 中，这里的的代码是不同的，在并发情况下会链表会变成环状，形成死锁。而JDK 8 已经修复了该问题，但是仍然不建议使用 HashMap 并发编程。

## **总结**
 
截至到这里，我们的 HashMap 的 put 方法已经剖析完了，此次可以说收获不小：

我们知道了，无论我们如何设置初始容量，HashMap 都会将我们改成2的幂次方，也就是说，HashMap 的容量百分之百是 2的幂次方，因为HashMap 太依赖他了。但是，请注意：如果我们预计插入7条数据，那么我们写入7，HashMap 会设置为 8，虽然是2的幂次方，但是，**请注意**，当我们放入第7条数据的时候，就会引起扩容，造成性能损失，所以，知晓了原理，我们以后在设置容量的时候还是自己算一下，比如放7条数据，我们还是都是设置成16，这样就不会扩容了。

HashMap 的默认加载因子是 0.75，虽然可以修改，但是出于安全考虑，除非你经过大量测试，请不要修改此值，HashMap 使用此值基本是平衡了性能和空间的取舍。

HashMap 扩容的时机是，容器中的元素数量  > 负载因此 * 容量，如果负载因子是0.75，容量是16，那么当容器中数量达到13 的时候就会扩容。还有，如果某个链表长度达到了8，并且容量小于64，则也会用扩容代替红黑树。

HashMap 在 JDK 7 中并发扩容的时候是非常危险的，非常容易导致链表成环状。但 JDK 8 中已经修改了此bug。但还是不建议使用。强烈推荐并发容器 ConcurrentHashMap。

HashMap 扩容的时候，不管是链表还是红黑树，都会对这些数据进行重新的散列计算，然后缩短他们的长度，优化性能。在进行散列计算的时候，会进一步优化性能，减少减一的操作，直接使用& 运算。可谓神来之笔。

总之，HashMap 中的优秀的设计思想值得我们去学习，最让楼主震惊的就是那个将任意一个数变成了2的幂次方的数，并且该数字很合理，说实话，如果让楼主写，楼主是写不出来的。

所以，请努力吧，现在做不到，不代表以后做不到，通过学习优秀的源码，一定能够提高我们的编码能力。


加油加油！！！！！！


对了，今天是 2017年的最后一天，明天就是2018 年了，借这篇文章祝大家新年快乐。每个人都能实现自己的新年愿望。

 
























