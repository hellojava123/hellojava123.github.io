---
layout: post
title: Java-中最大的数据结构：LinkedHashMap-了解一下？
date: 2018-05-19 11:11:11.000000000 +09:00
---
## 前言

Map 家族数量众多，其中 HashMap 和 ConcurrentHashMap 用的最多，而 LinkedHashMap 似乎则是不怎么用的，但是他却有着顺序。两种，一种是添加顺序，一种是访问顺序。

## 详情

LinkedHashMap 继承了 HashMap。那么如果是你，你怎么实现这两个顺序呢？

如果实现添加顺序的话，我们可以在该类中，增加一个链表，每个节点对应 hash 表中的桶。这样，循环遍历的时候，就可以按照链表遍历了。只是会增大内存消耗。

如果实现访问顺序的话，同样也可以使用链表，但每次读取数据时，都需要更新一下链表，将最近一次读取的放到链尾。这样也就能够实现。此时也可以跟进这个特性实现 LRU（Least Recently Used） 缓存。

## 如何使用？

下面是个小 demo

```java
LinkedHashMap<Integer, Integer> map = new LinkedHashMap<>(16, 0.75f, true);
for (int i = 0; i < 10; i++) {
  map.put(i, i);
}

for (Map.Entry entry : map.entrySet()) {
  System.out.println(entry.getKey() + ":" + entry.getValue());
}
map.get(3);
System.out.println();
for (Map.Entry entry : map.entrySet()) {
  System.out.println(entry.getKey() + ":" + entry.getValue());
}
```  

打印结果：

```java
0:0
1:1
2:2
3:3
4:4
5:5
6:6
7:7
8:8
9:9

0:0
1:1
2:2
4:4
5:5
6:6
7:7
8:8
9:9
3:3
```

首先构造方法是有意思的，比  HashMap 多了一个 accessOrder boolean 参数。表示，按照访问顺序来排序。最新访问的放在链表尾部。

如果是默认的，则是按照添加顺序，即 accessOrder 默认是 false。


## 源码实现

如果看 LinkedHashMap 内部源码，会发现，内部确实维护了一个链表：

```java
    /**
     * 双向链表的头，最久访问的
     */
    transient LinkedHashMap.Entry<K,V> head;

    /**
     * 双向链表的尾，最新访问的
     */
    transient LinkedHashMap.Entry<K,V> tail;
```

而这个 LinkedHashMap.Entry 内部也维护了双向链表必须的元素，before，after：

```java
    /**
     * HashMap.Node subclass for normal LinkedHashMap entries.
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

在添加元素的时候，会追加到尾部。

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }

    // link at the end of list
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
```

在 get 的时候，会根据 accessOrder 属性，修改链表顺序：

```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }

    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

同时注意：这里修改了 modCount，即使是读操作，并发也是不安全的。

## 如何实现  LRU 缓存？

LRU 缓存：`LRU（Least Recently Used，最近最少使用）算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。`

LinkedHashMap 并没有帮我我们实现具体，需要我们自己实现 。具体实现方法是 removeEldestEntry 方法。

一起来看看原理。

首先，HashMap 在 putVal 方法最后，会调用 afterNodeInsertion 方法，其实就是留给 LinkedHashMap 的。而 LinkedHashMap 的具体实现则是根据一些条件，判断是否需要删除 head 节点。

源码如下：

```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```

evict 参数表示`是否需要删除某个元素`，而这个 if 判断需要满足的条件如上：head 不能是 null，调用 removeEldestEntry 方法，返回 true 的话，就删除这个 head。而这个方法默认是返回 false 的，等待着你来重写。

所以，removeEldestEntry 方法的实现通常是这样：

```java
public boolean removeEldestEntry(Map.Entry<K, V> eldest){    
   return size() > capacity;  
} 
```

如果长度大于容量了，那么就需要清除不经常访问的缓存了。afterNodeInsertion 会调用 removeNode 方法，删除掉 head 节点 —— 如果 accessOrder 是 true 的话，这个节点就是最不经常访问的节点。

## 拾遗

LinkedHashMap 重写了一些 HashMap 的方法，例如 containsValue 方法，这个方法大家猜一猜，怎么重写比较合理？

HashMap 使用了双重循环，先循环外层的 hash 表，再循环内层的 entry 链表。性能可想而知。

但 LinkedHashMap 内部有个元素链表，直接遍历链表就行。相对而言而高很多。


```java
    public boolean containsValue(Object value) {
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
```

这也算一种空间换时间的策略吧。

get 方法当然也是要重写的。因为需要根据 accessOrder 更新链表。


## 总结

雪薇的总结的一下：

LinkedHashMap 内部包含一个双向链表维护顺序，支持两种顺序——添加顺序，访问顺序。

默认就是按照添加顺序来的，如果要改成访问顺序的话，构造方法中的 accessOrder 需要设置成 true。这样，每次调用  get 方法，就会将刚刚访问的元素更新到链表尾部。

关于 LRU，在accessOrder 为 true 的模式下，你可以重写 removeEldestEntry 方法，返回 `size() >  capacity `，这样，就可以删除最不常访问的元素。















