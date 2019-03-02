---
layout: post
title: 并发编程之-ThreadLocal-源码剖析
date: 2018-01-10 11:11:11.000000000 +09:00
---

![](http://upload-images.jianshu.io/upload_images/4236553-5dba534955b430d3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

首先看看 JDK 文档的描述：

> 该类提供了线程局部 (thread-local) 变量。这些变量不同于它们的普通对应物，因为访问某个变量（通过其 get 或 set 方法）的每个线程都有自己的局部变量，它独立于变量的初始化副本。ThreadLocal 实例通常是类中的 private static 字段，它们希望将状态与某一个线程（例如，用户 ID 或事务 ID）相关联。

> 每个线程都保持对其线程局部变量副本的隐式引用，只要线程是活动的并且 ThreadLocal 实例是可访问的；在线程消失之后，其线程局部实例的所有副本都会被垃圾回收（除非存在对这些副本的其他引用）

例如，以下类生成对每个线程唯一的局部标识符。 线程 ID 是在第一次调用 UniqueThreadIdGenerator.getCur

java.lang.ThreadLocal 不是 1.5 新加入的类，在 1.2 的时候就已经存在 java 类库的类，但该类的作用非常的大，所以我们也要剖析一下他的源码，也要验证关于该类的一些争论，比如内存泄漏。

## 1. 如何使用？

该类有4个方法方法需要关注：

```java
public class ThreadLocal<T> {

    public T get();

    public void set(T value);

    public void remove();

    protected T initialValue();
    
```

get() : 返回此线程局部变量的当前线程副本中的值。如果变量没有用于当前线程的值，则先将其初始化为调用  initialValue() 方法返回的值。


set()：将此线程局部变量的当前线程副本中的值设置为指定值。大部分子类不需要重写此方法，它们只依靠 initialValue() 方法来设置线程局部变量的值。

remove() ：移除此线程局部变量当前线程的值。如果此线程局部变量随后被当前线程读取， 且这期间当前线程没有设置其值，则将调用其 initialValue() 方法重新初始化其值。这将导致在当前线程多次调用 initialValue 方法。则不会对该线程再调用 initialValue 方法。通常，此方法对每个线程最多调用一次，但如果在调用 get() 后又调用了 remove（） ，则可能再次调用此方法。

 initialValue()：返回此线程局部变量的当前线程的“初始值”。线程第一次使用 get() 方法变量时将调用此方法，但如果线程之前调用了 set（T） 方法，


我们还是来个例子：

```java
package cn.think.in.java.lock.tools;

public class ThreadLocalDemo {

  public static void main(String[] args) {
    ThreadLocal<String> local = new MyThreadLocal<>();
    System.out.println(local.get());
    local.set("hello");
    System.out.println(local.get());
    local.remove();
    System.out.println(local.get());
  }

}

class MyThreadLocal<T> extends ThreadLocal<T> {

  @Override
  protected T initialValue() {
    return (T) "world";
  }
}


```
运行结果

```text
world
hello
world
```

上面的代码中，我们重写了 ThreadLocal  的 initialValue 方法，返回了一个字符串 “world”，第一次调用 get 方法返回了该值，而我们然后又调用 set 方法设置了 hello 字符串，再次调用 get 方法，此时返回的就是刚刚set 的值 ---- hello，然后我们调用remove 方法，删除 hello，再次调用 get 方法，返回了 initialValue 方法中的 world。

从这个流程中，我们已经知道了该类的用法，那么我们就看看源码是如何实现的。

## get() 源码剖析

```java
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 获取当前线程的 ThreadLocalMap  对象
        ThreadLocalMap map = getMap(t);
        // 如果map不是null，将 ThreadlLocal 对象作为 key 获取对应的值
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            // 如果该值存在，则返回该值
            if (e != null) {
                T result = (T)e.value;
                return result;
            }
        }
        // 如果上面的逻辑没有取到值，则从 initialValue  方法中取值
        return setInitialValue();
    }
```


楼主在该方法中写了注释，主要逻辑是从 当前线程中取出 一个类似 Map 的对象，
map 中 key是 ThreadLocal 对象，value 则是我们设置的值。如果 该 map中没有，则从 initialValue  方法中取。

我们继续看看，map 的真实面目：

![](http://upload-images.jianshu.io/upload_images/4236553-ef87994c62636ea3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4236553-a6d0201cef876e0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4236553-aea431b644513977.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

就是这个map，这个map 持有一个 Entry 数组，Entry 继承了 WeakReference ，也就是弱引用，如果一个对象具有弱引用，在GC线程扫描内存区域的过程中，不管当前内存空间足够与否，都会回收内存。这个特性我们之后再说。


总的来说，每个线程对象中都有一个 ThreadLocalMap 属性，该属性存储 ThreadLocal 为 key ，值则是我们调用 ThreadLocal 的 set 方法设置的，也就是说，一个ThreakLocal 对象对应一个 value。

还没完，我们看看 getEntry 方法：

```java
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```

如果hash 没有冲突，直接返回 对应的值，如果冲突了，调用 getEntryAfterMiss 方法。

getEntryAfterMiss 源码：
```java
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

```
该方法会循环所有的元素，直到找到 key 对应的 entry，如果发现了某个元素的 key 是 null，顺手调用 expungeStaleEntry 方法清理 所有 key 为 null 的 entry。 


那么 set 方法是怎么样的呢？

## 3. set() 方法源码剖析

源码如下：

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```




该方法同样先得到当前线程，然后根据当前线程得到线程的 ThreadLocalMap 属性，如果 Map 为null， 则创建一个Map ，并将值放置到Map中，否则，直接将值放置到Map中。

先看看 createMap(Thread t, T firstValue) 方法：

```java
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY]; // 默认长度16
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1); // 得到下标
            table[i] = new Entry(firstKey, firstValue); // 创建一个entry对象并插入数组
            size = 1; // 设置长度属性为1
            setThreshold(INITIAL_CAPACITY); 设置阀值== 16 * 2 / 3 == 10
        }

```

这个方法很简单，楼主已经写了详细的注释。就是创建一个16长度的entry 数组，设置阀值为10，注意，再resize 的时候，并不是10，而是 10 - 10 / 4，也就是 8，负载因子为 0.5，和 HashMap 是不同的。

我们再看看 map.set(ThreadLocal<?> key, Object value) 方法如何实现的：

```java
       private void set(ThreadLocal<?> key, Object value) {
            
            Entry[] tab = table;
            int len = tab.length;
            // 根据 ThreadLocal 的 HashCode 得到对应的下标
            int i = key.threadLocalHashCode & (len-1);
            // 首先通过下标找对应的entry对象，如果没有，则创建一个新的 entry对象
            // 如果找到了，但key冲突了或者key是null，则将下标加一（加一后如果小于数组长度则使用该值，否则使用0），
            // 再次尝试获取对应的 entry，如果不为null，则在循环中继续判断key 是否重复或者k是否是null
            for (Entry e = tab[i]; e != null;  e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                // key 相同，则覆盖 value
                if (k == key) {
                    e.value = value;
                    return;
                }
                // 如果key被 GC 回收了（因为是软引用），则创建一个新的 entry 对象填充该槽
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            
            // 创建一个新的 entry 对象
            tab[i] = new Entry(key, value);
            // 长度加一
            int sz = ++size;
            // 如果没有清楚多余的entry 并且数组长度达到了阀值，则扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

这里楼主刚开始有一个奇怪的地方，为什么这里和 HashMap 处理 Hash 冲突的方式不一样，楼主后来查询资料，才明白，HashMap 的Hash冲突方法是拉链法，即用链表来处理，而 ThreadLocalMap 处理Hash冲突采用的是线性探测法，即这个槽不行，就换下一个槽，直到插入为止。但是该方法有一个问题，就是，如果整个数组都冲突了，就会不停的循环，导致死循环，虽然这种几率很小。

我们继续往下。

如果 k == null，表示 ThreadLocal 被GC回收了，那么就调用 replaceStaleEntry 方法重新生成一个 entry，不过该方法没有我说的那么简单，我们来看看：

```java
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```

该方法可以说有点复杂，楼主看了很久，真的没想到 ThreadLocal 这么复杂。。。。。如同该方法名称，该方法会删除陈旧的 entyr，什么是陈旧的呢，就是 ThreadLocal 为 null 的 entry，会将 entry  key 为 null 的对象设置为null。核心的方法就是 expungeStaleEntry（int）；

整体逻辑就是，通过线性探测法，找到每个槽位，如果该槽位的key为相同，就替换这个value；如果这个key 是null，则将原来的entry 设置为null，并重新创建一个entry。

不论如何，只要走到了这里，都会清除所有的 key 为null 的entry，也就是说，当hash 冲突的时候并且对应的槽位的key值是null，就会清除所有的key 为null 的entry。

我们回到 set 方法。如果 hash 没有冲突，也会调用 cleanSomeSlots 方法，该方法同样会清除无用的 entry，也就是 key 为null 的节点。我们看看代码：

```java
      private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
```
该方法会遍历所有的entry，并判断他们的key，如果key是null，则调用 expungeStaleEntry 方法，也就是清除 entry。最后返回 true。

如果返回了 false ，说明没有清除，并且 size 还 大于等于 10 ，就需要 rahash，该方法如下：

```java
       private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }
```

首先会调用 expungeStaleEntries 方法，该方法会清除无用的 entry，我们之前说过了，同时，也会对 size 变量做减法，如果减完之后，size 还大于 8，则调用 resize 方法做真正的扩容。

resize 方法如下：

```java
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```

该方法会直接扩容为原来的2倍，并将老数组的数据都移动到 新数组，size 变量记录了里面有多少数据，最后设置扩容阀值为 2/3。

所以说，扩容分为2个步骤，当长度达到了容量的2/3，就会清理无用的数据，如果清理完之后，长度还大于等于阀值的3/4，那么就做真正的扩容。而不是网上很多人说的达到了 2/3 就扩容。这里的误区就是扩容之前需要清理。清理完之后再做判断。


可以看到，每次调用set 方法都会进行清理工作。实际上，如果使用 get 方法，当对应的 entry 的key为null 的时候，也会进行清理。

## 4. remove 方法源码剖析

```java
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```

通过线性探测法找到 key 对应的 entry，调用 clear 方法，将 ThreadLocal 设置为null，调用 expungeStaleEntry 方法，该方法顺便会清理所有的 key 为 null 的 entry。

## 5. Thread 线程退出时清理 ThreadLocal

Thread 的exit 方法：


```java
    private void exit() {
        if (group != null) {
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
    }
```

可以看到，该方法会将线程相关的所有属性变量全部清除。包括 threadLocals。

## 总结

楼主开始以为这个类的代码不会很难，想来楼主太天真了。从源码中我们可以看到，ThreadLocal 类的作者无时无刻都在想着如何去除那些 key 为 null 的 元素，为什么？因为只要线程不退出，这些变量都会一直留在线程中。

但是，Java 中有线程池的技术，也就是说，线程基本不会退出，因此，就需要手动去删除这些变量。如果你在线程中放置了一个大大的对象，使用完几次后没有清除（调用 remove 方法），该对象将会一直留在线程中。造成了内存泄漏。

为什么要使用弱引用呢？我们假设一下，不用弱引用，如果我们使用的 ThreadLocal 的变量是个局部变量，并设置到了线程中，当这个方法结束时，我们没有调用 remove 方法，而 Map 中 key 不是弱引用，那么该变量将会一直存在！！！

如果使用了弱引用，就算你没有调用 remove 方法，GC 也会清除掉 Map 中的引用，同时，ThreadLocal 也会通过对 key 是否为 null 进行判断，从而防止内存泄漏。

这里我们重新总结一下：ThreadLocal 的作者之所以使用弱引用，是担心程序员使用了局部变量的ThreadLocal 并且没有调用 remove 方法，这将导致没有结束的线程发生内存泄漏。使用弱引用，即使程序员没有删除，GC 也会将该变量设置为null，ThrealLocal 通过判断 key 是否为 null 来清除无用数据。防止内存泄漏。

当然，如果你使用的是静态变量，并且使用结束后没有设置为 null， ThrealLocal 是无法自动删除的，因此需要调用 remove 方法。

那么，ThrealLocal 什么时候会自动回收呢？当调用 remove 方法的时候（废话），当调用 get 方法并且 hash 冲突了的时候（情况很少），调用 set 方法时 hash 冲突了，调用 set 方法时正常插入。注意，调用 set 方法时，如果是覆盖操作，则不会执行清理。


我们正常使用 ThreadLocal 都是静态变量，也是 JDK 建议的例子，所以一定要手动调用 remove 方法，或者使用完毕后置为 null。反之，你可以碰运气不好，JDK 可能会帮你删，比如在你 set 的时候（也就是我们上面说的那几种情况），如果运气不好，就会永远存在线程中，导致内存泄漏。

所以，强烈建议手动调用 remove 方法。








