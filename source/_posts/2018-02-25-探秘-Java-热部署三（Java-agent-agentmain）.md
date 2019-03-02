---
layout: post
title: 探秘-Java-热部署三（Java-agent-agentmain）
date: 2018-02-25 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-337aa07271f40693.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

让我们继续探秘 Java 热部署。在前文 [探秘 Java 热部署二（Java agent premain）](https://www.jianshu.com/p/0bbd79661080)中，我们介绍了 Java agent premain。通过在main方法之前通过类似 AOP 的方式添加 premain 方法，我们可以在类加载之前做修改字节码的操作，无论是第一次加载，还是每次新的 ClassLoader 加载，都会经过 ClassFileTransformer 的 transform 方法，也就是说，都可以在这个方法中修改字节码，虽然他的方法名是 premain ，但是我们确实可以利用这个特性做这个事情。

在文章的最后，我们也提到了，虽然相比较在自定义类中修改字节码，premain 没有什么侵入性，对业务透明，但是美中不足的是，他还需要在启动的时候增加参数。

我们还提到了，premain 虽然可以热部署，但是还需要重新创建类加载器，虽然，这的确也符合 JVM 关于类的唯一性的定义。但是，有一种情况，如果使用的是系统类加载器，我们也无法创建新的ClassLoader对象。那么我们也就无法重新加载类了，怎么办呢？还好 Java 6 为我们提供了一种方法，也就是今天的主角 agentmain。

## 1. 什么是 agentmain？

和 premain 师出同门，我们知道，premain 只能在类加载之前修改字节码，类加载之后无能为力，只能通过重新创建ClassLoader 这种方式重新加载。而 agentmain 就是为了弥补这种缺点而诞生的。简而言之，agentmain 可以在类加载之后再次加载一个类，也就是重定义，你就可以通过在重定义的时候进行修改类了，甚至不需要创建新的类加载器，JVM 已经在内部对类进行了重定义（重定义的过程相当复杂）。

但是这种方式对类的修改是由限制的，对比原来的老类，由如下要求：

1.父类是同一个；
2. 实习那的接口数也要相同；
3. 类访问符必须一致；
4. 字段数和字段名必须一致；
5. 新增的方法必须是 private static/final 的；
6. 可以删除修改方法；


可以看的出来，相比较重新创建类加载器，限制还是挺多的，最重要的字段是无法修改的。因此，使用的时候要注意。

但是，agentmain 还有一个强大的特点，就是目标程序什么都不需要管，就能够被代理。还记得 premain 是如何使用的吗？需要在目标应用启动的时候增加 -javaagent 参数。虽说没有侵入性，但相比 agentmain 而言，还是有侵入性的，毕竟 agentmain 什么都不要。目标程序独立运行，什么都不用管。

那我们就来试试吧！

## 2. 如何使用？

agentmain 使用步骤如下：
1. 定义一个MANIFEST.MF 文件，文件中必须包含 Agent-Class;
2. 创建一个 Agent-Class 指定的类，该类必须包含 agentmain 方法（参数和 premian 相同）。
3. 将MANIFEST.MF 和 Agent 类打成 jar 包;
4. 将 jar 包载入目标虚拟机。目标虚拟机将会自动执行 agentmain 方法执行方法逻辑，同时，ClassFileTransformer 也会长期有效，在每一个类加载器加载 Class 的时候都会拦截。

让我们按着步骤来一遍吧！

1. 定义一个MANIFEST.MF 文件，文件中必须包含 Agent-Class;

````properties
Manifest-Version: 1.0
Can-Redefine-Classes: true
Agent-Class: cn.think.in.java.clazz.loader.asm.agent.AgentMainTraceAgent 
Can-Retransform-Classes: true
````

2. 创建一个 Agent-Class 指定的类，该类必须包含 agentmain 方法（参数和 premian 相同）。

````java
public class AgentMainTraceAgent {

  public static void agentmain(String agentArgs, Instrumentation inst)
      throws UnmodifiableClassException {
    System.out.println("Agent Main called");
    System.out.println("agentArgs : " + agentArgs);
    inst.addTransformer(new ClassFileTransformer() {

      @Override
      public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
          ProtectionDomain protectionDomain, byte[] classfileBuffer)
          throws IllegalClassFormatException {
          System.out.println("agentmain load Class  :" + className);
          return classfileBuffer;
      }
    }, true);
    inst.retransformClasses(Account.class);
  }
````

上面的类逻辑很简单，打印 Agent Main called，并打印参数，然后添加一个类转换器，转换器中的逻辑只是打印字符串，为了简单起见，并没有修改字节码（各位可自行使用ASM 等类库修改）。最后一点很重要，执行了 inst.retransformClasses(Account.class); 这段代码的意思是，重新转换目标类，也就是 Account 类。也就是说，你需要重新定义哪个类，需要指定，否则 JVM 不可能知道。还有一个类似的方法 redefineClasses ，注意，这个方法是在类加载前使用的。类加载后需要使用 retransformClasses 方法。

3. 将MANIFEST.MF 和 Agent 类打成 jar 包;
这个请自行 google。maven 或者 ide 都可以。
4. 将 jar 包载入目标虚拟机。目标虚拟机将会自动执行 agentmain 方法执行方法逻辑，同时，ClassFileTransformer 也会长期有效，在每一个类加载器加载 Class 的时候都会拦截。

这段代码很重要：

````java
class JVMTIThread {

  public static void main(String[] args)
      throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
    List<VirtualMachineDescriptor> list = VirtualMachine.list();
    for (VirtualMachineDescriptor vmd : list) {
      if (vmd.displayName().endsWith("AccountMain")) {
        VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
        virtualMachine.loadAgent("E:\\self\\demo\\out\\artifacts\\test\\test.jar ", "cxs");
        System.out.println("ok");
        virtualMachine.detach();
      }
    }
  }
}

````
注意：写这段代码的时候 IDE 可能提示找不到 jar 包，这时候将 jdk/lib/tools.jar 添加的项目的 classpath 中，具体请自行 google。
该main方法步骤如下：
1. 获取当前系统所有的虚拟机，类似 jps 命令。
2. 循环所有虚拟机，找到 AccountMain 虚拟机。
3. 将当前JVM 链接上目标JVM，并加载 loadAgent jar 包且传递参数。
4.  卸载虚拟机。

如何测试呢？

首先看测试类，也就是AccountMain类：

````java
class AccountMain {

  public static void main(String[] args) throws InterruptedException {
    for (;;) {
      new Account().operation();
      Thread.sleep(5000);
    }

  }
}

````

当我们一切准备就绪，启动 AccountMain 后，再启动 JVMTIThread 类，结果如下：

![运行结果](http://upload-images.jianshu.io/upload_images/4236553-1f2f3490a5aa6516.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看到，执行了1遍 operation方法后，我们启动了 attach jvm，随后，agentmain 方法就被执行了，并打印了我们我们传递的字符串参数，同时也进入到了 ClassFileTransformer 方法中，表示重新加载了 Account 类。有点遗憾的是，限于篇幅，我们没有修改字节码。不过楼主还是展示一下楼主修改字节码的结果吧，假设楼主想统计 operation 方法的时长，楼主将使用 ASM 增加一段统计时长的代码：

````java
public class TimeStat {
  static ThreadLocal<Long> threadLocal = new ThreadLocal<>();
  public static void start() {
    threadLocal.set(System.currentTimeMillis());
  }
  public static void end() {
    long time = System.currentTimeMillis() - threadLocal.get();
    System.out.print(Thread.currentThread().getStackTrace()[2] + "方法耗费时间: ");
    System.out.println(time + "毫秒");
  }
}

````
然后修改 agentmain 方法中ClassFileTransformer 的transform 逻辑，也就是在这里修改字节码。
运行结果如下：

![成功修改字节码](http://upload-images.jianshu.io/upload_images/4236553-bd3209028def98ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，首先重定义了 Account 类，又主动加载了 TimeStat 类，然后生效，在 operation 字符串后面打印了方法的耗时。


## 总结

通过对 agentmain 的使用，我们感受到了他的强大，在目标程序丝毫不改动的，甚至连启动参数都不加的情况下，可以修改类，并且是运行后修改，而且不重新创建类加载器。其主要得益于 JVM 底层的对类的重定义，关于底层代码解释，Jvm 大神寒泉子有篇文章 [JVM源码分析之javaagent原理完全解读](http://www.infoq.com/cn/articles/javaagent-illustrated) ，详细分析了 javaagent 的原理。但 agentmain 有一些功能上的限制，比如字段不能修改增减。所以，使用的时候需要权衡，到底使用哪种方式实现热部署。

说了这么久的热部署，其实就是动态或者说运行时修改类，大的方向说有2种方式：
1. 使用 agentmain，不需要重新创建类加载器，可直接修改类，但是有很多限制。
2. 使用 premain 可以在类第一次加载之前修改，加载之后修改需要重新创建类加载器。或者在自定义的类加载器种修改，但这种方式比较耦合。

无论是哪种，都需要字节码修改的库，比如ASM，javassist ，cglib 等，很多。总之，通过java.lang.instrument 包配合字节码库，可以很方便的动态修改类，或者进行热部署。

good luck！！！！！


参考: [JVM源码分析之javaagent原理完全解读](http://www.infoq.com/cn/articles/javaagent-illustrated)
《实战Java 虚拟机》
[javaagent](https://liuzhengyang.github.io/2017/03/15/javaagent/)
[Instrumentation 新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)









