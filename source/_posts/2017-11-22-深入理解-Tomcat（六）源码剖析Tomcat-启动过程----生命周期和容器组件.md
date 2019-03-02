---
layout: post
title: 深入理解-Tomcat（六）源码剖析Tomcat-启动过程----生命周期和容器组件
date: 2017-11-22 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-4ed959889f31f675.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

好了,今天我们继续分析 tomcat 源码, 这是第六篇了, 上一篇我们一边 debug 一边研究了 tomcat 的类加载体系, 我觉得效果还不错, 楼主感觉对 tomcat 的类加载体系的理解又加深了一点. 所以, 我们今天还是按照之前的方式来继续看源码, 一边 debug, 一边看, 今天我们分析的是tomcat 中2个非常重要的组件-------生命周期和容器. tomcat 庞大的架构, 他是如何管理每个对象的呢? 我们在[深入理解 Tomcat (二) 从宏观上理解 Tomcat 组件及架构](http://www.jianshu.com/p/d74eef07487f)中说过一段:

> 基于JMX Tomcat会为每个组件进行注册过程，通过Registry管理起来，而Registry是基于JMX来实现的，因此在看组件的init和start过程实际上就是初始化MBean和触发MBean的start方法，会大量看到形如： Registry.getRegistry(null, null).invoke(mbeans, "init", false); Registry.getRegistry(null, null).invoke(mbeans, "start", false); 这样的代码，这实际上就是通过JMX管理各种组件的行为和生命期。

当时大家可能还不是很理解这句话, 觉得这是在扯淡, 听不懂. 好吧, 今天我们就用代码说话, 看看 JMX 到底怎么管理 tomcat 的 组件.


### 1. 什么是 JMX?
我们之前说过:
> JMX 即 Java Management Extensions(JMX 规范), 是用来对 tomcat 进行管理的. tomcat 中的实现是 commons modeler 库, Catalina 使用这个库来编写托管 Bean 的工作. 托管 Bean 就是用来管理 Catalina 中其他对象的 Bean.

**简单来说: 就是一个可以为Java应用程序或系统植入远程管理功能的框架。**

既然是框架, 肯定要有架构图:
![](http://upload-images.jianshu.io/upload_images/4236553-f4eb03538b446841.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里对上图中三个分层进行介绍：

* Probe Level：负责资源的检测（获取信息），包含MBeans，通常也叫做Instrumentation Level。MX管理构件（MBean）分为四种形式，分别是标准管理构件（Standard MBean）、动态管理构件（Dynamic MBean）、开放管理构件(Open Mbean)和模型管理构件(Model MBean)。

* Agent Level：即MBeanServer，是JMX的核心，负责连接Mbeans和应用程序。 

* Remote Management Level：通过connectors和adaptors来远程操作MBeanServer，常用的控制台，例如JConsole、**VisualVM**(等会我们就要用这个)等。


### 2. 我们看看生命周期组件接口是如何设计的:


![](http://upload-images.jianshu.io/upload_images/4236553-d20b1a4da501035f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是一张 IDEA 生成的简单的 StandardHost(Host 容器的标准实现) 的 UML类图, 基本上, tomcat 的容器类都是这样的继承结构.

因此我们就可以直接看下面这张图:
![](http://upload-images.jianshu.io/upload_images/4236553-ebf5aa0870e5bf20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里对上图中涉及的主要类作个简单介绍：

1. Lifecycle：定义了容器生命周期、容器状态转换及容器状态迁移事件的监听器注册和移除等主要接口；

2. LifecycleBase：作为Lifecycle接口的抽象实现类，运用抽象模板模式将所有容器的生命周期及状态转换衔接起来，此外还提供了生成LifecycleEvent事件的接口；

3. LifecycleSupport：提供有关LifecycleEvent事件的监听器注册、移除，并且使用经典的监听器模式，实现事件生成后触达监听器的实现；

4. MBeanRegistration：Java JMX框架提供的注册MBean的接口，引入此接口是为了便于使用JMX提供的管理功能；

5. LifecycleMBeanBase：Tomcat提供的对MBeanRegistration的抽象实现类，运用抽象模板模式将所有容器统一注册到JMX；

6. 此外，ContainerBase、StandardServer、StandardService、WebappLoader、Connector、StandardContext、StandardEngine、StandardHost、StandardWrapper等容器都继承了LifecycleMBeanBase，因此这些容器都具有了同样的生命周期并可以通过JMX进行管理。

### 3. 再看看我们的容器结构

我们之前说, 如果从宏观上讲容器, 画画图, 讲讲就好了, 就可以在脑海里形成一个映象, 今天, 我们要好好的讲讲容器, 从代码层面去理解他们. 这样一来, 也顺便把我们的容器组件也讲了, 等于又讲了生命周期组件, 还有容器组件. 一举两得. 哈哈哈. 好吧, 不扯了, 回来, 我们继续讲容器. 还是先来一张图吧:

![](http://upload-images.jianshu.io/upload_images/4236553-5ca9ad548f0736fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图中我们可以看到:　StandardServer、StandardService、Connector、StandardContext这些容器，彼此之间都有父子关系，每个容器都可能包含零个或者多个子容器，这些子容器可能存在不同类型或者相同类型的多个. 所以他们都包含的关系, 如果让你来设计这些容器的生命周期, 你会用什么设计模式呢?

### 4. 容器初始化, 开始 Debug

首先我们启动 main 方法:
```java

    public static void main(String args[]) {
        try {
            // 命令
            String command = "start";
            // 如果命令行中输入了参数
            if (args.length > 0) {
                // 命令 = 最后一个命令
                command = args[args.length - 1];
            }
            // 如果命令是启动
            if (command.equals("startd")) {
                args[args.length - 1] = "start";
                daemon.load(args);
                daemon.start();
            }
            // 如果命令是停止了
            else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                daemon.stop();
            }
            // 如果命令是启动
            else if (command.equals("start")) {
                daemon.setAwait(true);// bootstrap 和 Catalina 一脉相连, 这里设置, 方法内部设置 Catalina 实例setAwait方法
                daemon.load(args);// args 为 空,方法内部调用 Catalina 的 load 方法.
                daemon.start();// 相同, 反射调用 Catalina 的 start 方法 ,至此,启动结束
            } else if (command.equals("stop")) {
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                daemon.load(args);
                if (null==daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable t) {
            // Unwrap the Exception for clearer error reporting
            if (t instanceof InvocationTargetException &&
                    t.getCause() != null) {
                t = t.getCause();
            }
            handleThrowable(t);
            t.printStackTrace();
            System.exit(1);
        }
    }
```
熟悉这个方法或者看过我们上篇文章的同学都知道, 我已经把类加载那部分代码去除了, 因为我们今天不研究类加载. 所以 ,我们看逻辑, 首先, 判断命令是什么, 我们现在的命令肯定是 start 啊, 所以进入 else if 块, 调用 load 方法 , 进入 load 方法, 可以看到, 该方法实际上就是 Catalina 类的 load 方法, 那么我们进入 Catalina 类的 load 方法看看(方法很长, 楼主去除了和今天的模块无关的代码):

```java
public void load() {
        // Start the new server
        try {
            getServer().init();
        } catch (LifecycleException e) {
            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                throw new java.lang.Error(e);
            } else {
                log.error("Catalina.start", e);
            }
        }
 }
```
可以看到, 这里有一个我们今天感兴趣的方法, getServer.init(), 这个方法看名字是启动 Server 的初始化, 而 Server 是我们上面图中最外层的容器. 因此, 我们去看看该方法, 也就是LifecycleBase.init() 方法. 该方法是一个模板方法, 只是定义了一个算法的骨架, 将一些细节算法延迟到了子类中. 看, 我们又学到了一个设计模式. 我们看看该方法:
```java
 @Override
    public final synchronized void init() throws LifecycleException {
        // 1
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
        }
        // 2
        setStateInternal(LifecycleState.INITIALIZING, null, false);

        try {
            // 模板方法
            /**
             * 采用模板方法模式来对所有支持生命周期管理的组件的生命周期各个阶段进行了总体管理，
             * 每个需要生命周期管理的组件只需要继承这个基类，
             * 然后覆盖对应的钩子方法即可完成相应的声明周期阶段的管理工作
             */
            initInternal();
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            setStateInternal(LifecycleState.FAILED, null, false);
            throw new LifecycleException(
                    sm.getString("lifecycleBase.initFail",toString()), t);
        }

        // 3
        setStateInternal(LifecycleState.INITIALIZED, null, false);
    }

```

我们看看该方法, 这应该就是容器启动的逻辑了, 先前我们定义了那么多状态, 现在用上了. 首先判断该方法的状态, 如果不是 NEW, 则抛出异常， 否则则设置状态为 INITIALIZING， 然后调用一个抽象方法 initInternal , 该方法由子类具体实现. 执行完则修改状态为 INITIALIZED. 这里应该是使用了状态模式. 依赖状态时,同步该方法, 防止并发错误. tomcat 可以的.

### 5. 那么我们来看看 StandardServer 是如何实现 initInternal 方法的:

```java
    @Override
    protected void initInternal() throws LifecycleException {
        
        super.initInternal();

        // Register global String cache
        // Note although the cache is global, if there are multiple Servers
        // present in the JVM (may happen when embedding) then the same cache
        // will be registered under multiple names
        onameStringCache = register(new StringCache(), "type=StringCache");

        // Register the MBeanFactory
        MBeanFactory factory = new MBeanFactory();
        factory.setContainer(this);
        onameMBeanFactory = register(factory, "type=MBeanFactory");
        
        // Register the naming resources
        globalNamingResources.init();
        
        // Populate the extension validator with JARs from common and shared
        // class loaders
        if (getCatalina() != null) {
            ClassLoader cl = getCatalina().getParentClassLoader();
            // Walk the class loader hierarchy. Stop at the system class loader.
            // This will add the shared (if present) and common class loaders
            while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
                if (cl instanceof URLClassLoader) {
                    URL[] urls = ((URLClassLoader) cl).getURLs();
                    for (URL url : urls) {
                        if (url.getProtocol().equals("file")) {
                            try {
                                File f = new File (url.toURI());
                                if (f.isFile() &&
                                        f.getName().endsWith(".jar")) {
                                    ExtensionValidator.addSystemResource(f);
                                }
                            } catch (URISyntaxException e) {
                                // Ignore
                            } catch (IOException e) {
                                // Ignore
                            }
                        }
                    }
                }
                cl = cl.getParent();
            }
        }
        // Initialize our defined Services
        for (int i = 0; i < services.length; i++) {
            services[i].init();
        }
    }
```
### 6. LifecycleMBeanBase.initInternal() 实现

首先调用父类的  super.initInternal() 方法，此initInternal方法用于将容器托管到JMX，便于运维管理：
```java
    @Override
    protected void initInternal() throws LifecycleException {
       
        // If oname is not null then registration has already happened via
        // preRegister().
        if (oname == null) {
            mserver = Registry.getRegistry(null, null).getMBeanServer();
            oname = register(this, getObjectNameKeyProperties());
        }
    }

```
### 7. LifecycleMBeanBase.register 方法实现

LifecycleMBeanBase 会调用自身的 register 方法， 该方法会将容器注册到 MBeanServer：


```java
    protected final ObjectName register(Object obj,
            String objectNameKeyProperties) {
        
        // Construct an object name with the right domain
        StringBuilder name = new StringBuilder(getDomain());
        name.append(':');
        name.append(objectNameKeyProperties);

        ObjectName on = null;

        try {
            on = new ObjectName(name.toString());
            // 核心实现：registerComponent
            Registry.getRegistry(null, null).registerComponent(obj, on, null);
        } catch (MalformedObjectNameException e) {
            log.warn(sm.getString("lifecycleMBeanBase.registerFail", obj, name),
                    e);
        } catch (Exception e) {
            log.warn(sm.getString("lifecycleMBeanBase.registerFail", obj, name),
                    e);
        }

        return on;
    }
    
```

该方法内部核心方法是 Registry. registerComponent, 在org.apache.catalina.util 包下， 我们看看该方法实现。

### 8. Registry.registerComponent 方法实现
```java
  public void registerComponent(Object bean, ObjectName oname, String type)
           throws Exception
    {
        if( log.isDebugEnabled() ) {
            log.debug( "Managed= "+ oname);
        }

        if( bean ==null ) {
            log.error("Null component " + oname );
            return;
        }

        try {
            if( type==null ) {
                type=bean.getClass().getName();
            }

            ManagedBean managed = findManagedBean(bean.getClass(), type);

            // The real mbean is created and registered
            DynamicMBean mbean = managed.createMBean(bean);

            if(  getMBeanServer().isRegistered( oname )) {
                if( log.isDebugEnabled()) {
                    log.debug("Unregistering existing component " + oname );
                }
                getMBeanServer().unregisterMBean( oname );
            }

            getMBeanServer().registerMBean( mbean, oname);
        } catch( Exception ex) {
            log.error("Error registering " + oname, ex );
            throw ex;
        }
    }
```
该方法会为当前容器创建一个 DynamicMBean ， 并且注册到MBeanServer。调用 MBeanServer.registerMBean() 方法。而 MBeanServer 在 javax.management， 也就是 rt.jar 中，该包由 java 的 BootStrap 启动类加载器加载。

注册进MBeanServer 的 key 是什么呢？ 相信细心的同学会注意到 LifecycleMBeanBase.getObjectNameKeyProperties 和 LifecycleMBeanBase.getDomain 方法 和
LifecycleMBeanBase.getDomainInternal 方法， 这三个方法由具体子类实现，会生成一个专属于容器的key。格式为：`Catalina:type=Server`， 这是 Server 容器的 key， debug 可以看出来：

![](http://upload-images.jianshu.io/upload_images/4236553-fd0ee1be841208a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 9. JMX 如何管理 组件？ 

至此， 我们已经知道 Tomcat 是如何将容器注册到 MBeanServer 中的。 那么注册到 MBeanServer 中后是什么样子呢？我们看图:

![](http://upload-images.jianshu.io/upload_images/4236553-c1765c2cd2d6c9b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是 JDK 自带的 JvisualVM 工具， 添加了 MBeans 插件， 就可以远程操作容器中的 组件了， 可以看到 Service 容器暴漏了很多接口， 用于运维人员管理容器和组件。

### 10. 回到 StandardServer.initInternal 方法

好了， 我们回到 StandardServer.initInternal 方法， 回到我们梦最开始的地方，` super.initInternal` 方法就是将容器注册到 JMX 中。 那下面的逻辑是做什么的呢？ 在执行完父类的 super.initInternal 的方法后， 该方法又注册个两个 JMX 。然后寻启动子容器的 init 方法：
```java
    // Initialize our defined Services
        for (int i = 0; i < services.length; i++) {
            services[i].init();
        }
```

而子容器的 init 方法和 Server 的 init 方法的逻辑基本一致，所以不再赘述。

### 11. 执行完 ` getServer().init()` 方法后做什么------容器启动

Bootstrap 的 load 方法调用了 Catalina 的 load 方法 ，该方法调用了Server 的init方法，执行完初始化过程，当然就是要执行 start 方法了， 那么如何执行呢？

 Bootstrap 调用了 Catalina 的 start 方法，该方法也同样执行了 Server 的 start 方法， 该方法的具体实现也在LifecycleBase 中：
```java
@Override
    public final synchronized void start() throws LifecycleException {
        
        if (LifecycleState.STARTING_PREP.equals(state) ||
                LifecycleState.STARTING.equals(state) ||
                LifecycleState.STARTED.equals(state)) {
            
            if (log.isDebugEnabled()) {
                Exception e = new LifecycleException();
                log.debug(sm.getString("lifecycleBase.alreadyStarted",
                        toString()), e);
            } else if (log.isInfoEnabled()) {
                log.info(sm.getString("lifecycleBase.alreadyStarted",
                        toString()));
            }
            
            return;
        }
        
        if (state.equals(LifecycleState.NEW)) {
            init();
        } else if (state.equals(LifecycleState.FAILED)){
            stop();
        } else if (!state.equals(LifecycleState.INITIALIZED) &&
                !state.equals(LifecycleState.STOPPED)) {
            invalidTransition(Lifecycle.BEFORE_START_EVENT);
        }

        setStateInternal(LifecycleState.STARTING_PREP, null, false);

        try {
            startInternal();
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            setStateInternal(LifecycleState.FAILED, null, false);
            throw new LifecycleException(
                    sm.getString("lifecycleBase.startFail",toString()), t);
        }

        if (state.equals(LifecycleState.FAILED) ||
                state.equals(LifecycleState.MUST_STOP)) {
            stop();
        } else {
            // Shouldn't be necessary but acts as a check that sub-classes are
            // doing what they are supposed to.
            if (!state.equals(LifecycleState.STARTING)) {
                invalidTransition(Lifecycle.AFTER_START_EVENT);
            }
            
            setStateInternal(LifecycleState.STARTED, null, false);
        }
    }
```

### 12. StandardServer.startInternal 启动容器方法实现

可以看到该方法对状态的判断特别多，我们感兴趣的是 try 块中的  startInternal() 方法， 同样， 该方法也是个抽象方法，需要子类去具体实现自己的启动逻辑。我们看看Server 的启动逻辑：
```java
 @Override
    protected void startInternal() throws LifecycleException {

        fireLifecycleEvent(CONFIGURE_START_EVENT, null);
        setState(LifecycleState.STARTING);//将自身状态更改为LifecycleState.STARTING；
        globalNamingResources.start();
        // Start our defined Services
        synchronized (services) {
            for (int i = 0; i < services.length; i++) {
                services[i].start();// 启动所有子容器
            }
        }
    }
```

### 13. LifecycleSupport.fireLifecycleEvent()方法实现

该方法首先执行自己的`fireLifecycleEvent`方法， 该方法内部是LifecycleSupport.fireLifecycleEvent()方法,  我们进入该方法看个究竟:

```java
public void fireLifecycleEvent(String type, Object data) {
        // 事件监听,观察者模式的另一种方式
        LifecycleEvent event = new LifecycleEvent(lifecycle, type, data);
        LifecycleListener interested[] = listeners;// 监听器数组 关注 事件(启动或者关闭事件)
        // 循环通知所有生命周期时间侦听器????
        for (int i = 0; i < interested.length; i++)
            // 每个监听器都有自己的逻辑
            interested[i].lifecycleEvent(event);
    }
```

该方法很简单, 楼主没有删一行代码, 首先, 创建一个事件对象, 然通知所有的监听器发生了该事件.并做响应.那么 Server 有哪些监听器呢?

![](http://upload-images.jianshu.io/upload_images/4236553-7b9d96095834dd0e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这些监听器将根据这个事件的类型做出响应. 

### 14. 我们回到 startInternal 方法, 启动所有容器

事件监听结束之后,  调用 `setState(LifecycleState.STARTING);` 表明状态时开始中, 并且循环启动子容器, 这里的 Server 启动的是Service 数组, 循环启动他们的 start 方法. 以此类推. 启动所有的容器：
```java
    synchronized (services) {
            for (int i = 0; i < services.length; i++) {
                services[i].start();// 启动所有子容器
            }
        }
```
现在我们关注的是Server 容器， 因此， Server 会启动 services 数组中的所有 Service 组件。该方法就完成了通知所有监听器发送了启动事件，然后使用观察者模式，启动所有子容器，然后子容器继续递归启动。最后修改自己的状态并告诉监听器。

### 15. 总结

其实楼主在啃代码最深的感触就是设计模式， 真的很牛逼，不知道同学们发现了几个设计模式，楼主在本篇文章中发现了状态模式， 观察者模式，模板方法， 事件监听，代理模式。真的收益良多。不枉楼主每天看代码。

还有就是对 Tomcat 生命周期组件的总结。我们再看看我们的类图：

![](http://upload-images.jianshu.io/upload_images/4236553-ebf5aa0870e5bf20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

tomcat 的主要容器都继承了 LifecycleMBeanBase 抽象类，该类中关于 init 和 start 两个模板方法。定义了主要算法骨架，而方法中又都有抽象方法，需要子类自己去实现。而 LifecycleBase 中又定义了如何实现事件监听代理，LifecycleBase 依赖 LifecycleSupport 去完成真正的事件监听。对了，监听器是如何添加进 LifecycleSupport 的呢？LifecycleSupport 中含有
 addLifecycleListener 方法。该方法也是被LifecycleBase代理的。而每个容器下面的子容器也是使用相同的逻辑完成初始化和启动。父容器和子容器使用了聚合的方式设计。

可以说， tomcat的容器的生命周期组件设计是非常牛逼的。我们阅读源码不仅能了解他的设计原理，也能同大师交流，学会更多。

好了， 今天的深入理解 Tomcat（六）源码剖析Tomcat 启动过程----生命周期和容器组件就到这里，谢谢大家的耐心，再这个世界，耐心和坚持是无比珍贵的。尤其是程序员。

good luck ！！！！
