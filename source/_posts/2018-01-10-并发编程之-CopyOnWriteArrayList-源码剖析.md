---
layout: post
title: 并发编程之-CopyOnWriteArrayList-源码剖析
date: 2018-01-10 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-2f54843f43d60a5d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

ArrayList 是一个不安全的容器，在多线程调用 add 方法的时候会出现 ArrayIndexOutOfBoundsException 异常，而 Vector 虽然安全，但由于其 add 方法和 get 方法都使用了 synchronized 关键字，导致在并发时的性能令人担忧，因此，伟大的 Doug Lea 编写了 CopyOnWriteArrayList 并发容器，用于替代并发时的 ArrayList，而该类的类名叫 “写的时候拷贝集合”。也非常符合他的设计，那么，我们就看看他是如何实现的。

## 源码剖析

既然是 ArrayList ，第一个看的当然是 add 方法。

add 方法源码

```java
      public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
该方法步骤如下：
1. 使用重入锁锁住代码块。
2. 调用 getArray 方法获取当前数组，调用 Arrays 工具类的 copyof 方法，在原来数组长度的基础上加一创建一个新数组，然后将元素添加到新数组的最后以为，这点和 ArrayList 不同， ArrayList 需要扩容的时候在原有的基础上扩容一半。
3. 调用 setArray 方法，将新数组赋值到成员变量 array 中，注意：该变量是 volatile 的。因此，其他线程可以立即看到他。最后释放锁。

再看看 get 方法：

```java
   public E get(int index) {
        return get(getArray(), index);
    }
    private E get(Object[] a, int index) {
        return (E) a[index];
    }

```

可以看到该方法非常简单：获取到成员变量 array，根据下标获取。没有使用锁，完全支持并发。

从 add 方法和 get 方法中，我们看出了作者的意图，Doug Lea 认为容器 get 的操作比 add 的操作频繁，使用了类似读写分离的方式，读读操作完全并发，而写的时候，并不修改原有的内容，这对于保证当前在读线程的数据一致性非常重要，然后对原有的数据进行一次复制，将改写的内容写入到副本中，写完之后，再将修改完的副本替换原来的数据。这样就可以保证写操作不会影响读了。同时使用 volatile 变量，也保证了内存可见性，更新之后立即就能被其他线程看到。

该类保证了并发时的安全，同时，相比于 Vector 性能要高出很多（读读完全并发）。而 Vector 同时只能有一个线程进行读写，简直可怕。

但该类也不是完美的。该类的迭代器不支持 remove 操作，也不支持 set 操作，也不支持 add 操作。在迭代器中调用这 三个方法将抛出 UnsupportedOperationException 异常。

但是该类可以在 for 循环中做删除操作，这点和 ArrayList 也是不一样的，因为该类每次删除之后也都是 拷贝重写赋值。而 ArrayList 使用 for 循环删除实际上使用的迭代器的 next 方法，而迭代器每次都会检查ArrayList 的状态，调用 checkForComodification 方法，因此会抛出 ConcurrentModificationException 异常。ArrayList 必须使用迭代器的 remove 方法。

```java
        public void remove() {
            throw new UnsupportedOperationException();
        }


        public void set(E e) {
            throw new UnsupportedOperationException();
        }


        public void add(E e) {
            throw new UnsupportedOperationException();
        }
```


## 总结

总的来说，并发环境下，强烈建议使用该类代替 ArrayList，该类的读读操作可保证完全并发。支持 for 循环做删除操作。不支持迭代器remove ，set， add。






























