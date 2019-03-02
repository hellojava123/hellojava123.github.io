---
layout: post
title: 深入浅出-JVM-ClassLoader
date: 2018-02-21 11:11:11.000000000 +09:00
---

## # 前言

在 JVM 综述里面，我们说，JVM 做了三件事情，Java 程序的内存管理，
Java Class 二进制字节流的加载（ClassLoader），Java 程序的执行（执行引擎）。我们也说，我们大部分情况下只关注前2个。在前面的文章中，我们已经分析了内存关系相关的，包括运行时数据区，GC 相关。今天我们要讲的就是类加载器。

在 [JVM 综述](https://www.jianshu.com/p/70154dc5a9ff) 里，我们已经大致分析了一些概念。而今天的文章将详细的阐述类加载器。

首先，我们要了解类加载器，当然，了解的目的是为了更好的开发，通过对类加载器的解读，看看我们能不能做些什么，比如修改类加载器的加载逻辑，比如加入自定义的类加载器等等功能。

让我们开始吧！


## # 1.  类加载器介绍

对于 Java 虚拟机来说，Class 文件是一个重要的接口，无论使用何种语言进行软件开发，只要能将源文件编译为正确的 Class 文件，那么这种语言就可以在 Java 虚拟机上运行。可以说，Class 文件就是虚拟机的基石。

如图所示：

![各种语言都可以在 JVM 上运行](http://upload-images.jianshu.io/upload_images/4236553-0462adabdf0aac40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上图可以看出，虚拟机不拘泥于 Java 语言，任何一个源文件只要能编译成 Class 文件的格式，就可以在JVM 上运行！Class 文件格式就像是一个接口，只要遵守这个接口，就能够在 JVM 上运行。


## # 2. 类加载器的工作流程

Class 文件通常是以文件的方式存在（任何二进制流都可以是 Class 类型），但只有能被 JVM 加载后才能被使用，才能运行编译后的代码。系统装在 Class 类型可以分为加载，链接和初始化三个步骤。其中，链接也可分为验证，准备和解析3步骤。如图所示：

![Class 文件转载过程](http://upload-images.jianshu.io/upload_images/4236553-b58c0180435f2e21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中，只有加载过程是程序员能够控制的，后面的几个步骤都是有虚拟机自动运行的。因此，我们的关注点主要放在加载阶段。



## # 3.  类加载流程中的 “加载”

上面说了，类加载器3个流程中，唯一能让程序员 “做手脚” 的就是加载过程，上面是加载过程呢？其主要作用就是从系统外部获得 Class 二进制数据流。

JVM 不会无故装载 Class 文件，只有在必要的时候才装载，哪几个时候呢？
1. 当创建一个类的实例是，比如使用 new 关键字，或者通过反射，克隆，反序列化。
2. 当调用类的静态方法时，即当使用字节码 invokstatic 指令。
3. 当使用类或接口的静态字段时（final 常量除外），比如，使用 getstatic 或者 pustatic 指令。
4. 当时用 Java.lang.reflect 包中的方法反射类的方法时。
5. 当初始化子类，要求先初始化父类。
6. 作为启动虚拟机，含有 main（）方法的那个类。

以上6种情况属于主动调用，主动调用会触发初始化，还有一种情况是被动调用，则不会引起初始化。

#### # 3.1 ClassLoader 抽象类介绍

Java 类加载器的具体实现就在 java.lang.ClassLoader，该类是一个抽象类，并且提供了一些重要的接口，用于自定义Class 的加载流程和加载方式。主要方法如下：

1. public Class<?> loadClass(String name) throws ClassNotFoundException
给定一个类名，加载一个雷，返回代表这个类的 Class 实例，如果找不到类，则返回异常。

2. protected final Class<?> defineClass(String name, byte[] b, int off, int len) throws ClassFormatError
根据给定的字节码流 b 定义一个类，off 表示位置，len 表示长度。该方法只有子类可以使用。

3. protected Class<?> findClass(String name) throws ClassNotFoundException
查找一个类，也是只能子类使用，这是重载 ClassLoader 时，最重要的系统扩展点。这个方法会被 loadClass 调用，用于自定义查找类的逻辑，如果不需要修改类加载默认机制，只是想改变类加载的形式，就可以重载该方法。

4. protected final Class<?> findLoadedClass(String name)
同样的，这个方法也只有子类能够使用，他会去寻找已经加载的类，这个方法是 final 方法，无法被修改。

同时，在该类中，还有一个字段非常重要：parent，他也是一个 ClassLoader 的实例，这个字段所表示的 ClassLoader 也称为这个 ClassLoader 的双亲，在类加载的过程中，ClassLoader 可能会将某些请求交给自己的双亲处理。

#### # 3.2 类加载器的双亲委派模型

在标准的 Java 程序中，从虚拟机的角度讲，只有2种类加载器：
1. 启动类加载器（BootStrap ClassLoader），C++ 语言实现，虚拟机自身的一部分
2. 另一种就是所有其他的类加载器，由 Java 语言实现，独立于虚拟机外部，并且全部继承自抽身类 java.lang.ClassLoader。

从程序员的角度讲，虚拟机会创建 3 中类加载器，分别是：Bootstrap ClassLoader(启动类加载器），Extension ClassLoader（扩展类加载器）和 APPClassLoader（应用类加载器，也称为系统类加载器）。此外，每一个应用程序还可以拥有自定义的 ClassLoader，扩展 Java 虚拟机获取 Class 数据的能力。

而这 3 个类加载器有着层次关系。

先来看一个著名的图：

![类加载器双亲委派模型](http://upload-images.jianshu.io/upload_images/4236553-eba420d0b83f65a1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如图所示：从 ClassLoader 的层次自顶向下为启动类加载器，扩展类加载器，应用类加载器和自定义类加载器，当系统需要适用一个类时，在判断类是否已经被加载时，会先从当前底层类加载器进行判断，但系统需要加载一个类时，会从顶层类开始加载，依次向下尝试，直到成功。

注意，我们无法访问启动类加载器，当试图获取启动类加载器的时候，返回 null，因此，如果返回的是 null，并不意味没有类加载器为它服务，而是指哪个类为启动类加载器。

那么这些类加载路径是哪些呢？
1. BootStrap 类加载器负责加载 <JAVA_HOME>/lib 目录中的，或者别-Xbootclasspath 参数指定的路径。并且是被虚拟机识别的，如 rt.jar，名字不符合的类库即使放在 lib 目录中也不会加载。

2. 扩展类加载器有 sun.misc.Launcher$ExtClassLoader 实现，负责加载 <JAVA_HOME>/lib/ext 目录中的。或者被 java.ext.dirs 系统变量所指定的路径中的所有类库。

3. 应用类加载器由 sun.misc.Launcher$AppClassLoader 实现，由于这个类是 ClassLoader 中的 getSystemClassLoader 方法的返回值，也称为系统类加载器，负载加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器。一般情况下，这个就是程序中默认的类加载器。

4. 自定义类加载器用于加载一些特殊途径的类，一般也是用户程序类。

系统中的 ClassLoader 在协同工作时，默认会使用双亲委托模式，即在类加载的时候，系统会判断当前类是否已经被加载，如果已经加载，则直接返回，否则就尝试加载，在尝试加载时，会先请求双亲处理，如果双亲查找事变，则自己加载。代码如下：

````java
 protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

````

代码中，如果双亲是 null，则使用启动类加载器加载，如果事变，则使用当前类加载器加载。

双亲为 null 一般有2种情况，1. 双亲是启动类加载器。2. 自己就是启动类加载器。

其中加载类的逻辑有2个注意的地方。

1. 判断是否已经加载？当判断类是否需要加载时，是从底层开始判断，如果底层已经加载了，则不再请求双亲。

2. 当系统准备加载一个类时。会先从双亲加载，也就是最顶层的启动类加载器，逐层向下，直到找到该类。和上面的是相反的。


#### # 3.3 类加载器的双亲委派模型缺陷和补充

双亲模型固然有着优点，能够让整个系统保持了类的唯一性。但在有些场合，却不适合，也就是说，顶层的启动类加载器的代码无法访问到底层的类加载器。如 rt.jar 无法中代码无法访问到应用类加载器。

你肯定要问，为什么需要访问呢？

在 Java 平台中，把核心类（rt.jar)中提供外部服务，可由应用层自行实现的接口，通常可以称为 Service Provider Interface，即 SPI。

在 rt.jar 中的抽象类需要加载继承他们的在应用层的子类实现，但是以目前的双亲机制是无法实现的。

因此 JDK 引用了一个不太优雅的设计，上下文类加载器。也就是讲类加载放在线程上下文变量中。通过     Thread.getContextClassLoader()，    Thread.setContextClassLoader(ClassLoader) 这两个方法获取和设置 ClassLoader，这样，rt.jar 中的代码就可以获取到底层的类加载了。

#### # 3.4 突破双亲模式

双亲模式是虚拟机的默认行为，但并非必须这么做，通过重载 ClassLoader 可以修改该行为。事实上，很多框架和软件都修改了，比如 Tomcat，OSGI。具体实现则是通过重写 loadClass 方法，改变类的加载次序。比如先使用自定义类加载器加载，如果加载不到，则交给双亲加载。

## # 4. 类加载的扩展---热替换

我们知道：由不同的 ClassLoader 加载的同名类属于不同的类型，不能相互转化和兼容。

而这个特性就是我们实现热替换的关键。过程如图所示：

![热替换基本思路](http://upload-images.jianshu.io/upload_images/4236553-dbe5357dffc7d83b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## # 总结

好了，到这里，基本的类加载器就介绍结束了。我们总结了类加载的工作流程，包括加载，连接，初始化。然后我们重点介绍了加载，因为加载阶段是我们程序员唯一有所作为的地方。然后介绍了加载阶段的一些细节，比如双亲委派，然后说了双亲委派的缺点和补充，然后探讨了如何修改默认的类加载方式，最后通过类加载的特性实现了热替换。当然也看了核心类 ClassLoader 的源码。不过，这肯定不是类加载器的全部。我们将在后面的文章中将类加载的其他特性一一解开。

good luck！！！！

