---
layout: post
title: 探秘-Java--热部署二（Java-agent-premain）
date: 2018-02-25 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-d678a3abf7a261af.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## # 前言

在前文 [探秘 Java 热部署](https://www.jianshu.com/p/731bc8293365) 中，我们通过在死循环中重复加载 ClassLoader 和 Class 文件实现了热部署的功能，但我们也指出了缺点-----不够灵活。需要手动修改文件等操作。

如果有那么一种功能，当你需要重新加载类并修改类的时候，有那么一个转换器自动帮你修改已有的 Class 文件变成你设定的 Class 文件，那么就不需要手动修改编译了。

也许你第一想到的就是在自定义类加载器中做文章，比如在 loadClass 中，得到字节码之后，通过 ASM 或者 javassist 修改字节码，然后再调用 defineClass 方法。

确实可行，但是这种侵入性太大。如果JVM 在底层提供一种类似 “类转换器” 的东西，是不是侵入性就不大了呢？

实际上，JVM 确实给我们提供了一个工具，那就是今天的主角------java agent。

## 1. 什么是 Java agent

在 JDK 1.5 中，Java 引入了 java.lang.Instrument 包，该包提供了一些工具帮助开发人员在 Java 程序运行时，动态修改系统中的 Class 类型。其中，使用该软件包的一个关键组件就是 Java  agent。从名字上看，似乎是个 Java 代理之类的，而实际上，他的功能更像是一个Class 类型的转换器，他可以在运行时接受重新外部请求，对Class 类型进行修改。

如果在命令行执行 java 命令，会出现一些命令帮助，其中就有 java agent的选项：

![java 命令](http://upload-images.jianshu.io/upload_images/4236553-7cae3d3d7a6cab7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2. Java agent 详细介绍

参数 javaagent 可以用于指定一个 jar 包，并且对该 java 包有2个要求：
1. 这个 jar 包的MANIFEST.MF 文件必须指定 Premain-Class 项。
2. Premain-Class 指定的那个类必须实现 premain（）方法。

重点就在 premain 方法，也就是我们今天的标题。从字面上理解，就是运行在 main 函数之前的的类。当Java 虚拟机启动时，在执行 main 函数之前，JVM 会先运行 -javaagent 所指定 jar 包内 Premain-Class 这个类的 premain 方法，其中，该方法可以签名如下：

1.public static void premain(String agentArgs, Instrumentation inst)
2.public static void premain(String agentArgs)

JVM 会优先加载 1 签名的方法，加载成功忽略 2，如果1 没有，加载 2 方法。这个逻辑在sun.instrument.InstrumentationImpl 类中：

![InstrumentationImpl  类](http://upload-images.jianshu.io/upload_images/4236553-37cfc8c3c42f0986.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


参数 agentArgs 时通过命令行传给 Java  Agent 的参数， inst 是 Java Class 字节码转换的工具，Instrumentation 常用方法如下：

1.  void addTransformer(ClassFileTransformer transformer, boolean canRetransform);
增加一个Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。

2. void  redefineClasses(ClassDefinition... definitions) hrows  ClassNotFoundException, UnmodifiableClassException;
在类加载之前，重新定义 Class 文件，ClassDefinition 表示对一个类新的定义，如果在类加载之后，需要使用 retransformClasses 方法重新定义。

3.  boolean removeTransformer(ClassFileTransformer transformer);
删除一个类转换器

4.  void  retransformClasses(Class<?>... classes) throws UnmodifiableClassException
在类加载之后，重新定义 Class。这个很重要，该方法是1.6 之后加入的，事实上，该方法是 update 了一个类。

## 3. 如何使用？

使用 javaagent 需要几个步骤：
1. 定义一个 MANIFEST.MF 文件，必须包含 Premain-Class 选项，通常也会加入Can-Redefine-Classes 和 Can-Retransform-Classes 选项。
2. 创建一个Premain-Class 指定的类，类中包含 premain 方法，方法逻辑由用户自己确定。
3. 将 premain 的类和 MANIFEST.MF 文件打成 jar 包。
4. 使用参数 -javaagent:/jar包路径=[agentArgs 参数] 启动要代理的方法。

在执行以上步骤后，JVM 会先执行 premain 方法，大部分类加载都会通过该方法，注意：是大部分，不是所有。当然，遗漏的主要是系统类，因为很多系统类先于 agent 执行，而用户类的加载肯定是会被拦截的。

也就是说，这个方法是在 main 方法启动前拦截大部分类的加载活动，注意：是类加载之前。也就是说，我们可以在这个缝隙中做很多文章，比如修改字节码。

让我们来试试：

1 首先定义一个 MANIFEST.MF 文件:

````properties
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Premain-Class: cn.think.in.java.clazz.loader.asm.agent.PreMainTraceAgent
````

2. 创建一个Premain-Class 指定的类，类中包含 premain 方法：
````java
public class PreMainTraceAgent {

  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("agentArgs : " + agentArgs);
    inst.addTransformer(new ClassFileTransformer() {
      @Override
      public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
          ProtectionDomain protectionDomain, byte[] classfileBuffer)
          throws IllegalClassFormatException {
        System.out.println("premain load Class     :" + className);
        return classfileBuffer;
      }
    }, true);
  }
}

````

3. 将 premain 的类和 MANIFEST.MF 文件打成 jar 包 . 
使用 IDEA 的 build ，当然你也可以使用 maven。具体请 google。

4. 使用参数 -javaagent:/jar包路径=[agentArgs 参数] 启动要代理的方法。
我们当然需要一个测试类：

````java

class AccountMain {


  public static void main(String[] args)
      throws ClassNotFoundException, InterruptedException, IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
    while (true) {
     Account account = new Account();
     account.operation();
    }
  }

}

class Account {
  public void operation() {
    System.out.println("operation....");
  }
}

````

VM 参数 -javaagent:/jar包路径=[agentArgs 参数] 。

运行结果：
![执行结果](http://upload-images.jianshu.io/upload_images/4236553-0d5f689e096d69f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，我们的 premain 方法确实在 main 方法之前被调用了，并且是在类加载的时候被调用的，而我们重写的 transform 方法其中的 classfileBuffer 参数就是即将被载入虚拟机的字节码，因此，我们可以使用各种字节码库进行修改。具体修改，这里就暂时不表，以后有机会好好写写。

## 4. 和热部署有什么关系？

说了这么多，Java agent 确实可以在 main 方法之前并加载字节码的同时进行代理，有点类似 AOP。但到底和热部署有什么关系呢？

回忆刚开始我们说的，我们如果自己自定义一个类加载器，那么就可以在重新加载类（新的类加载器）的时候对字节码进行修改，但是对业务代码侵入性较大，如果在底层，也就是 JVM 层面，在加载字节码的时候回调某个方法，在该方法中修改字节码，岂不是达到了我们的目的？

我们再看看 premain 方法：

````java
  public static void premain(String agentArgs, Instrumentation inst) {
    System.out.println("agentArgs : " + agentArgs);
    inst.addTransformer(new ClassFileTransformer() {
      @Override
      public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
          ProtectionDomain protectionDomain, byte[] classfileBuffer)
          throws IllegalClassFormatException {
        System.out.println("premain load Class     :" + className);
        return classfileBuffer;
      }
    }, true);
  }
````

该方法中的 Instrumentation 添加了一个类转换器，该转换器是长期有效的，当该转换器被添加之后，只要有类加载的活动，都会被拦截。假设，我们的业务是当某个类需要修改，我们就重新加载（重新创建类加载器的前提）原来的字节码，加载之后，注意：加载之后对该字节码进行修改。而这些操作对业务代码来说，完全是透明的，基本没有侵入性（加入了 VM 参数）。

## 总结

通过上面的步骤，我们将 [探秘 Java 热部署](https://www.jianshu.com/p/731bc8293365) 的代码进行了优化，原本是直接手动修改字节码，现在通过加载原来的字节码，在原来的字节码基础上进行修改，再重新加载，完成了一次热部署。

而着一切得益于 java agent，虽然使用自定义的类加载也可以做到，但是似乎显得不是很优雅。使用 java agent 能让修改字节码这个动作化于无形，对业务透明，减少侵入性。

实际上，premain 还是有缺点的。什么缺点？竟然还要再命令行加参数？能不能不加参数？可以！ java 1.6 已经为我们准备了这个工具。也就是 agentmain ，可以不加任何参数，就可以修改一个类，甚至不需要重新创建类加载器！神奇吗？我们说，一个类加载器只能加载一个类，要想修改一个类，必须重新创建一个新的类加载器。但是，JVM 为我们做了很多，他在底层直接修改了类定义。使得我们不必重新创建类加载器了。具体我们将在下篇文章中详细介绍。

good luck！！！！！











