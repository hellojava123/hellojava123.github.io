---
layout: post
title: 深入理解-Tomcat（五）源码剖析Tomcat-启动过程----类加载过程
date: 2017-11-21 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-3ef513e8de839335.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是我们深入理解tomcat的第五篇文章，按照我们的思路，这次我们本应该区分析tomcat的连接器组件，但楼主思前想后，觉得连接器组件不能只是纸上谈兵，需要深入源码，但楼主本能的认为我们应该先分析tomcat的启动过程，以能够和我们上一篇文章`深入理解 Tomcat（四）Tomcat 类加载器之为何违背双亲委派模型`相衔接。因为启动类加载器的核心代码就在启动过程中，所以，我决定先分析tomcat的启动过程，结合源码了解tomcat的类加载器如何实现，以彻底了解tomcat的类加载器。

因为Tomcat 的启动过程非常复杂，因此楼主将启动过程拆开分析，不能像之前说的按照启动过程分析了，否则，文章篇幅过长，条理也会变得不清晰，Tomcat的启动过程包括了初始化容器的生命周期，还涉及到JMX的管理，还有我们现在分析的类加载器，因此，我们必须换个维度分析。

再一个，因为连接器和容器紧紧关联，连接器的作用就是分析http请求，所以，楼主觉得我们之前的计划可能需要变更一下，我们将在分析完生命周期和类加载器之后将结合源码分析连接器和容器，以了解tomcat的核心组件在接受HTTP请求后如何运行。

所以，今天，我们的任务就是debug tomcat 源码，分析tomcat启动过程的每一步操作。在看这篇文章之前希望同学们看看我们的第四篇分析tomcat的文章以了解tomcat的类加载器。

下面是这篇文章的目录结构：
```
1. 启动tomcat，进入main方法
2. 进入 init 方法
3. 进入 setCatalinaHome 方法
4. 进入 setCatalinaBase 方法
5. 接下来则是类加载器大显身手的时候了. 进入 intiClassLoaders 方法
6. 进入 createClassLoader 方法
7. 我们回到 initClassLoaders 方法中来
8. 再回到 init 方法中, 类加载器初始化结束, 接下来干嘛?
9. 设置完线程上下文类加载器之后做什么呢? 进入 securityClassLoad 方法
10. 进入 loadCorePackage 方法
11. 回到 init 方法
12. 寻找 WebAppClassLoader， 进入 startInternal 方法
13. 进入 createClassLoader 方法
14. tomcat 类加载结构体系创建完毕
```
### 1. 启动tomcat，进入main方法


我们打开我们之前clone下的Tomcat-Source-Code 源码，找到Bootstrap 类，找到main方法，在451行打上断点，启动main方法，开始我们的调试：![](http://upload-images.jianshu.io/upload_images/4236553-60673beedbf0dde4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到楼主已经写了很多的注释，因为楼主已经debug过了.
 我们看代码，首先判断”守护“对象是否为null，肯定为null了，然后进入if 块， 创建一个默认构造器的Bootstrap对象，有一行注释`// Don't set daemon until init() has completed`, 说不要在init方法完成之前设置daemon 变量，因为后面的很多步骤都依赖该变量，所以必须初始化结束后才能设置值，再继续看，进入init方法：

### 2. 进入 init 方法

![](http://upload-images.jianshu.io/upload_images/4236553-c23ee6115c29e76e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法注释：` Initialize daemon.`表明要初始化该守护程序，也就是这个变量Bootstrap：![](http://upload-images.jianshu.io/upload_images/4236553-1188f45052549f24.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看看该方法，首先`setCatalinaHome()`方法，也就是我们启动虚拟机的时候设置的 VM 参数:
![](http://upload-images.jianshu.io/upload_images/4236553-3286c5cedfea4fb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 我们进入该方法看看
### 3. 进入 setCatalinaHome 方法
![](http://upload-images.jianshu.io/upload_images/4236553-9f66c38cb434d70d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 很明显,我们设置过 Catalina.home , 所以获取classpath下的catalina.home 的值不为null，所以直接return， 如果不为null，则从项目根目录下获取boostrap的jar包。如果存在,则设置上一级目录为 catalina.home, 如果不存在,则设置项目根目录为 catalina.home.这就是 setCatalinaHome 方法的逻辑.

### 4. 进入 setCatalinaBase 方法
下一步是执行 setCatalinaBase 方法, 也是一样能获取`catalina.base`, 直接 return,如果不存在, 则设置` catalina.home `为`catalina.base`, 如果` catalina.home`也是空,那么则设置项目根目录为` catatlina.base`.

![](http://upload-images.jianshu.io/upload_images/4236553-e93ec0d70e3cb1a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
***
### 5. 接下来则是类加载器大显身手的时候了. 进入 intiClassLoaders 方法
![](http://upload-images.jianshu.io/upload_images/4236553-d24e1e66bf0ec5ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

intiClassLoaders(), 初始化多个类加载器. 我们进入该方法查看具体逻辑:

![](http://upload-images.jianshu.io/upload_images/4236553-a591da8d1f6fd60f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先,创建一个 common 类加载器, 父类加载器为 null, 注意: 如果是 java 推荐的类加载器机制, 父类加载器应该是系统类加载器或者是扩展类加载器, 所以这明显违背了类加载器的双亲委派模型. 好, 我们继续看 tomcat, 我们进入 creatClassLoader 方法, 看看是如何实现的(该方法很长, 我们关注重要的逻辑):

### 6.  进入 createClassLoader 方法
```java
private ClassLoader createClassLoader(String name, ClassLoader parent)
        throws Exception {
        // 从/org/apache/catalina/startup/catalina.properties 中获取该 key 的配置
        // common.loader 对应的 Value=${catalina.base}/lib,${catalina.base}/lib/*.jar,${catalina.home}/lib,${catalina.home}/lib/*.jar
        String value = CatalinaProperties.getProperty(name + ".loader");
        // 如果不存在(默认存在), 返回 null
        if ((value == null) || (value.equals(""))){
            return parent;
        }
        // 使用环境变量对应的目录替换字符串
        value = replace(value);

        //Repository是ClassLoaderFactory  中的一个静态内部类, 有2个属性, location, type, 表示某个位置的某种类型的文件
        List<Repository> repositories = new ArrayList<Repository>();
        /*
        省略部分代码, 此部分代码是处理value 变量并获取文件位置和类型
        */  

        // 根据给定的路径数组前去加载给定的 class 文件,
        // StandardClassLoader 继承了 java.net.URLClassLoader ,使用URLClassLoader的构造器构造类加载器.
        // 根据父类加载器是否为 null, URLClassLoader将启用不同的构造器.
        // 总之, common 类加载器没有指定父类加载器,违背双亲委派模型
        ClassLoader classLoader = ClassLoaderFactory.createClassLoader// 创建类加载器, 如果parent 是0,则
            (repositories, parent);

        // Retrieving MBean server // 注册到 JMX 中管理 bean, 这个我们后面说
        MBeanServer mBeanServer = null;
        if (MBeanServerFactory.findMBeanServer(null).size() > 0) {
            mBeanServer = MBeanServerFactory.findMBeanServer(null).get(0);
        } else {
            mBeanServer = ManagementFactory.getPlatformMBeanServer();
        }

        // Register the server classloader
        ObjectName objectName =
            new ObjectName("Catalina:type=ServerClassLoader,name=" + name);
        // 生命周期管理.
        mBeanServer.registerMBean(classLoader, objectName);

        return classLoader;

    }
````
虽然上面的代码写了注释,但是我们还是梳理一下逻辑:
1. 从`/org/apache/catalina/startup/catalina.properties `配置文件中获取 lib 仓库和 jar 包位置.  如果key （分别为common.loader、server.loader、shared.loader）所对应的value 不存在则返回父类加载器. 默认情况有.则继续下面的逻辑.
2. 处理从配置文件中获取的字符串, 得到jar 包的位置.
3. 使用`ClassLoaderFactory.createClassLoader(repositories, parent)`方法,  该方法使用一个继承自`java.net.URLClassLoader`废弃的`StandardClassLoader`类加载 jar 处理后得到的文件, 实际上调用的是URLClassLoader的构造方法, 下面代码是 crateClassLoader 方法中最后的逻辑, array 中是 jar 包的地址,根据父类加载器是否为 null 调用`StandardClassLoader`不同的构造方法.
4. 将ClassLoader 注册到JMX服务中（这里涉及到生命周期的内容，我们按下不表，后面再说）。

![](http://upload-images.jianshu.io/upload_images/4236553-4adc8d0b36c05114.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面的是`StandardClassLoader`的构造方法,可以看到,实际上只是使用URLClassLoader 的构造方法:

![](http://upload-images.jianshu.io/upload_images/4236553-973fd96d79b6594d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而 URLClassLoader  将会调用父类 SecureClassLoader 的构造方法, 而 SecureClassLoader 将会调用 ClassLoader 的构造方法, 从而完成一个类加载器的初始化. 可谓不易.

##### 7. 我们回到 initClassLoaders 方法中来

执行完`commonLoader = createClassLoader("common", null);`, 接下里就会判断返回的类加载器是否为 null, 什么时候会为 null 呢? 找不到配置文件中的 key 的时候或者 key 对应的 value 为空的时候回返回 null, 如果返回 null, 那么就设置默认的类加载器为 common 类加载器. 

继续初始化 catalinaloader 和 sharedLoader 两个类加载器, 同样调用 createClassLoader 方法, 不同的是, 他们的父类加载器不是 null, 而是上面的 common 类加载器. 我们看看他们进入这个方法的时候回怎么样?我们 debug 看一下:
![](http://upload-images.jianshu.io/upload_images/4236553-fd0d2db752af5723.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4236553-e81758f680ec5dff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到两个获取到都是空, 所以他们直接返回父类加载器, 也就是说, 他们三个使用的是同一个类加载器.

### 8. 再回到 init 方法中, 类加载器初始化结束, 接下来干嘛?

![](http://upload-images.jianshu.io/upload_images/4236553-774ddc72bda64f5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将 catalinaLoader 类加载器设置为当前线程上下文类加载器. 并设置线程安全类加载器.同时检查是否安全, 如果不安全,则直接结束.

![](http://upload-images.jianshu.io/upload_images/4236553-05cbefb1693430cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 9. 设置完线程上下文类加载器之后做什么呢? 进入 securityClassLoad 方法
![](http://upload-images.jianshu.io/upload_images/4236553-a88757e3f188b647.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/4236553-7b0ecfa890763864.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，该类时加载Tomcat容器中类资源，传递的ClassLoader时catalinaLoader，也就是说，Tomcat容器的类资源都是catalinaLoader加载完成的。

securityClassLoad方法主要加载Tomcat容器所需的class，包括：
  * Tomcat核心class，即org.apache.catalina.core路径下的class；
  * org.apache.catalina.loader.WebappClassLoader$PrivilegedFindResourceByName；
  * Tomcat有关session的class，即org.apache.catalina.session路径下的class；
  * Tomcat工具类的class，即org.apache.catalina.util路径下的class；
  * javax.servlet.http.Cookie；
  * Tomcat处理请求的class，即org.apache.catalina.connector路径下的class；
  * Tomcat其它工具类的class，也是org.apache.catalina.util路径下的class；

#### 10. 进入 loadCorePackage 方法

我们以加载Tomcat核心class的loadCorePackage方法为例，我们看源码实现：
```java
  private static final void loadCorePackage(ClassLoader loader)
        throws Exception {
        final String basePackage = "org.apache.catalina.core.";
        loader.loadClass
            (basePackage +
             "AccessLogAdapter");
        loader.loadClass
            (basePackage +
             "ApplicationContextFacade$1");
        loader.loadClass
            (basePackage +
             "ApplicationDispatcher$PrivilegedForward");
        loader.loadClass
            (basePackage +
             "ApplicationDispatcher$PrivilegedInclude");
        loader.loadClass
            (basePackage +
            "AsyncContextImpl");
        loader.loadClass
            (basePackage +
            "AsyncContextImpl$DebugException");
        loader.loadClass
            (basePackage +
            "AsyncContextImpl$1");
        loader.loadClass
            (basePackage +
            "AsyncContextImpl$PrivilegedGetTccl");
        loader.loadClass
            (basePackage +
            "AsyncContextImpl$PrivilegedSetTccl");
        loader.loadClass
            (basePackage +
            "AsyncListenerWrapper");
        loader.loadClass
            (basePackage +
             "ContainerBase$PrivilegedAddChild");
        loader.loadClass
            (basePackage +
             "DefaultInstanceManager$1");
        loader.loadClass
            (basePackage +
             "DefaultInstanceManager$2");
        loader.loadClass
            (basePackage +
             "DefaultInstanceManager$3");
        loader.loadClass
            (basePackage +
             "DefaultInstanceManager$AnnotationCacheEntry");
        loader.loadClass
            (basePackage +
             "DefaultInstanceManager$AnnotationCacheEntryType");
        loader.loadClass
            (basePackage +
             "ApplicationHttpRequest$AttributeNamesEnumerator");
    }
```

我们可以看到，catalinaClassLoader 加载了该包下的类。根据我们之前的理解：catalinaClassLoader 加载的类是Tomcat容器私有的类加载器，加载路径中的class对于Webapp不可见。

#### 11. 回到 init 方法

![](http://upload-images.jianshu.io/upload_images/4236553-a0dc0cc7c09d64a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先打印一波日志, 然后使用类 catalinaLoader 类加载器加载 `org.apache.catalina.startup.Catalina` 类, 接着创建该类的一个对象, 名为 startupInstance, 意为"启动对象实例", 然后使用反射调用该实例的setParentClassLoader 方法, 参数为 sharedLoader, 表示该实例的父类加载器为 sharedLoader.



最后, 设置 catalinaDaemon 为该实例。

#### 12. 寻找 WebAppClassLoader， 进入 startInternal 方法

我们通过源码知道了初始化了commonClassLoader, catalinaClassLoader, sharedLoader，但是，我们想起我们上一篇文章的图，好像少了点什么？
![](http://upload-images.jianshu.io/upload_images/4236553-89bacc3467d513f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/462)

WebAppClassLoader呢？

我们知道， WebAppClassLoaser 是各个Webapp私有的类加载器，加载路径中的class只对当前Webapp可见，那么他是如何初始化的呢？WebAppClassLoaser 的初始化时间和这3个类加载器初始化的时间不同，由于WebAppClassLoaser 和Context 紧紧关联，因此咋初始化
 org.apache.catalina.core.StandardContext 会一起初始化 WebAppClassLoader， 该类中startInternal方法含有初始化类加载器的逻辑，核心源码如下：
```java
    @Override
    protected synchronized void startInternal() throws LifecycleException {
         if (getLoader() == null) {
              WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
              webappLoader.setDelegate(getDelegate());
              setLoader(webappLoader);
        }
        if ((loader != null) && (loader instanceof Lifecycle)) {
              ((Lifecycle) loader).start();
        }
    }
```

首先创建 WebAppClassLoader ， 然后 setLoader（webappLoader），再调用start方法，该方法是个模板方法，内部有 startInternal 方法用于子类去实现， 我们看WebAppClassLoader的startInternal 方法核心实现：
```java
    @Override
    protected void startInternal() throws LifecycleException {
            classLoader = createClassLoader();
            classLoader.setResources(container.getResources());
            classLoader.setDelegate(this.delegate);
            classLoader.setSearchExternalFirst(searchExternalFirst);
            if (container instanceof StandardContext) {
                classLoader.setAntiJARLocking(
                        ((StandardContext) container).getAntiJARLocking());
                classLoader.setClearReferencesStatic(
                        ((StandardContext) container).getClearReferencesStatic());
                classLoader.setClearReferencesStopThreads(
                        ((StandardContext) container).getClearReferencesStopThreads());
                classLoader.setClearReferencesStopTimerThreads(
                        ((StandardContext) container).getClearReferencesStopTimerThreads());
                classLoader.setClearReferencesHttpClientKeepAliveThread(
                        ((StandardContext) container).getClearReferencesHttpClientKeepAliveThread());
            }

            for (int i = 0; i < repositories.length; i++) {
                classLoader.addRepository(repositories[i]);
            }

    }
```
#### 13.  进入 createClassLoader 方法

首先`classLoader = createClassLoader();`创建类加载器，并且设置其资源路径为当前Webapp下某个context的类资源。最后我们看看createClassLoader的实现：
```java

    /**
     * Create associated classLoader.
     */
    private WebappClassLoader createClassLoader()
        throws Exception {

        Class<?> clazz = Class.forName(loaderClass);
        WebappClassLoader classLoader = null;

        if (parentClassLoader == null) {
            parentClassLoader = container.getParentClassLoader();
        }
        Class<?>[] argTypes = { ClassLoader.class };
        Object[] args = { parentClassLoader };
        Constructor<?> constr = clazz.getConstructor(argTypes);
        classLoader = (WebappClassLoader) constr.newInstance(args);

        return classLoader;

    }
```
这里的loaderClass 是 字符串 `org.apache.catalina.loader.WebappClassLoader`, 首先通过反射实例化classLoader。现在我们知道了， WebappClassLoader 是在 StandardContext 初始化的时候实例化的，也证明了WebappClassLoader 和 Context 息息相关。

#### 14. omcat 类加载结构体系创建完毕

至此，我们的Tomcat 类加载结构体系创建完毕。真TMD复杂啊！！！ 不过，请记住， 阅读源码是提高我们水平最快速的手段。源码中大师们的设计模式和各种高级用法。会让我们功力大增。继续加油吧！！！













