---
layout: post
title: 深入理解Spring-之-Spring-进阶开发必知必会-之-Spring-扩展接口
date: 2017-12-13 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-0b7501acaaed2f97.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##  # 前言

我们在前几篇文章中已经深入了解了 Spring 的 IOC 机制和 AOP 机制，在阅读源码的同时，楼主对 Spring 中设计模式的运用可以说五体投地，还有我们还知道更重要的一点就是：Spring 留给了我们大量的扩展接口供开发者去自定义自己的功能，甚至于 AOP 就是在 Spring 预留的扩展接口中实现的，意思是只要基于 Spring IOC，遵守 Spring 对扩展接口的约定，那么就能实现自己想要的功能。可见 IOC 的强大，那么。今天我们就将 Spring 留给我们的接口拿出来说一说。而我们的标题是Spring 进阶开发，为什么这么说，如果说只是简单的使用 Spring 中的bean，那么只是Spring的初级开发者。如何精通Spring 就看有没有掌握好Spring留给我们的这些扩展接口，以及如何使用他们。

我们今天主要讲述以下几个接口，如有遗漏，请指出：
1. FactroyBean 我们熟悉的AOP基础bean
2. BeanPostProcess 在每个bena初始化成前后做操作。
3. InstantiationAwareBeanPostProcessor 在Bean实例化前后做一些操作。
4. BeanNameAware、ApplicationContextAware 和 BeanFactoryAware 针对bean工厂，可以获取上下文，可以获取当前bena的id。
5. BeanFactoryPostProcessor Spring允许在Bean创建之前，读取Bean的元属性，并根据自己的需求对元属性进行改变，比如将Bean的scope从singleton改变为prototype。
6. InitialingBean 在属性设置完毕后做一些自定义操作 DisposableBean 在关闭容器前做一些操作。

## 1. FactroyBean 我们熟悉的AOP基础bean

这个Bean 我们再属性不过，我们再学习 AOP 的时候，知道 XML 方式的 AOP 就是通过该接口实现的。我们复习以下该接口的结构。

```java
public interface FactoryBean<T> {
	T getObject() throws Exception;
	Class<?> getObjectType();
	boolean isSingleton();
}

```

该接口定义了3个方法，获取bean实例，获取bean类型，是否是单例。Spring 在 IOC 初始化的时候，一般的Bean都是直接调用构造方法，而如果该Bean实现了FactoryBean 接口，则会调用该Bean的 getObject 方法获取bean，这也是Spring 使用此接口构造AOP的原因。在 IOC 调用此方法的时候，返回一个代理，完成AOP代理的创建。

我们做个测试：定义一个FactoryBean，重写他的 getObject 方法：

![](http://upload-images.jianshu.io/upload_images/4236553-0690073c1a323270.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

测试类中调用：

![](http://upload-images.jianshu.io/upload_images/4236553-420c6f81201b7dda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意：此时返回的已经是 AOP 类型的Bean，因为我们在 getObject 返回到是 new Aop（），验证了我们之前说的。使用该接口，能够为我们做很多有趣的事情。就靠你来想象了。

## 2. BeanPostProcess 在每个bean初始化成前后做操作。

该接口我们应该也非常的熟悉，还记的我们的注解配置的AOP是如何实现的。就是间接实现了该接口。在 IOC 初始化的时候，会调用的该接口的后置处理方法。我们看看该接口定义：

```java
public interface BeanPostProcessor {

	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}

````

一个是前置方法，一个是后置方法，注解方式的AOP的实现就是在 postProcessAfterInitialization 方法中实现的。我们写一个测试类，看看结果是什么？

![](http://upload-images.jianshu.io/upload_images/4236553-d5b8cb9a8e309d0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们在Bean 初始化之前后之后都打印以下该Bean的名称，那么运行结果是什么呢？下面是SpringBoot 启动后的一部分日志：

![](http://upload-images.jianshu.io/upload_images/4236553-2994bfb922c69ae6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到我们项目中所有的Bean在初始化的时候都调用该方法。因此，我们在以后的开发中就可以做一些自定义的事情。


## 3. InstantiationAwareBeanPostProcessor 在Bean实例化前后做一些操作。

这个接口实际上我们也是非常的熟悉，该接口在我们剖析注解配置AOP的时候是我们的老朋友，实际上，注解配置的AOP是间接实现 BeanPostProcess  接口的，而 InstantiationAwareBeanPostProcessor 就是继承该接口的。我们看看他的继承图谱：

![](http://upload-images.jianshu.io/upload_images/4236553-74dcb5b9177f71b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到该接口在继承的基础上又增加了3个方法，增加了扩展bean的功能。我们写个 Demo 测试一下:

![](http://upload-images.jianshu.io/upload_images/4236553-29ddb24f49ffece8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，需要实现 5 个方法，其中2个方法是 BeanPostProcess 接口定义的方法：在bean初始化的前后执行，而 InstantiationAwareBeanPostProcessor 则新增了 3 个方法，分别是 postProcessBeforeInstantiation （实例化之前），postProcessAfterInstantiation （实例化之后），postProcessPropertyValues （在处理Bean属性之前），开发者可以在这三个方法中添加自定义逻辑，比如AOP。我们看看运行结果。

![](http://upload-images.jianshu.io/upload_images/4236553-d96008103fc9befc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，所有的Bean在IOC的时候都执行了我们的方法，其中实例化在初始化之前执行，这个顺序对我们使用该接口是很重要的，千万不要弄混。

## 4. BeanNameAware、ApplicationContextAware 和 BeanFactoryAware 针对bean工厂，可以获取上下文，可以获取当前bena的id。

这三个接口的功能其中都一样，我们看看他们的继承图谱就知道了：

![](http://upload-images.jianshu.io/upload_images/4236553-f5fb19b0b26f3952.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，这三个接口都继承自 Aware 接口，并分别定义了自己的接口定义方法。实现这些接口就能得到Spring的Bean 工厂。从而调用getBean方法获取Bean。很多项目中都使用此接口做了Spring的工具类。比如可以像这么使用：

![](http://upload-images.jianshu.io/upload_images/4236553-244d60d8fba15307.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们写了一个测试类，然后看以下运行结果：

![](http://upload-images.jianshu.io/upload_images/4236553-d92ed33522990841.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行结果：

![](http://upload-images.jianshu.io/upload_images/4236553-dee8507b0239efd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可以看到，在IOC的过程中，该Bean的三个方法都被执行，我们就可以获取到容器，从而可以做很多自定义的额事情。

## 5. BeanFactoryPostProcessor Spring允许在Bean创建之前，读取Bean的元属性，并根据自己的需求对元属性进行改变，比如将Bean的scope从singleton改变为prototype。

我们看看该接口的定义：

![](http://upload-images.jianshu.io/upload_images/4236553-68faef012c4f21e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只定义了一个方法，该方法注释：在它的标准初始化之后修改应用程序上下文的内部bean工厂。所有的bean定义都已经加载了，但是还没有实例化bean。这允许覆盖或添加属性，甚至是对初始化bean的属性。参数是什么呢？应用程序上下文所使用的bean工厂。也就是说，我们可以获取某个Bean的定义，然后修改该Bean的定义：比如下面这样：

![](http://upload-images.jianshu.io/upload_images/4236553-181c10f1798c536a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看看运行结果：

![](http://upload-images.jianshu.io/upload_images/4236553-b0b2ed0df773fd6d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们将成功的单例的Bean改成了多例。

## 6. InitialingBean 在属性设置完毕后做一些自定义操作。 DisposableBean 在关闭容器前做一些操作。

我们写以下demo 看看是如何运行的：

实现这两个接口： 

![](http://upload-images.jianshu.io/upload_images/4236553-f8c60f69a5d149a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

启动类：

![](http://upload-images.jianshu.io/upload_images/4236553-9e5c0949f89c1dff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在运行启动类之后，就会关闭容器。退出虚拟机，我们看看运行结果：


![](http://upload-images.jianshu.io/upload_images/4236553-89fe607165ba9df0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在执行Set 属性方法后，立即执行 afterPropertiesSet 方法，因此，我们就可以在该方法中做一些事情，然后在执行  System.exit(0) 后，执行 destroy 方法，我们也可以在该方法中执行一些逻辑。


## 7 总结

我们了解了 Spring 留给我们的扩展接口，以提高我们使用 Spring 的水平，在以后的业务中，也就可以基于 Spring 做一些除了简单的注入这种基本功能之外的功能。同时，我们也发现，Spring 的扩展性非常的高，符合设计模式中的开闭原则，对修改关闭，对扩展开放，实现这些的基础就是 Spring 的 IOC，IOC 可以说是 Spring 的核心， 在 IOC 的过程中，对预定义的接口做了很多的预留工作。这让其他框架与 Spring 的组合变得更加的简单，我们在以后的开发工作中也可以借鉴 Spring 的思想，让程序更加的优美。




























































