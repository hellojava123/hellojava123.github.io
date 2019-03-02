---
layout: post
title: 深入理解Spring-之-源码剖析AOP（注解方式）
date: 2017-12-09 11:11:11.000000000 +09:00
---
上一篇文章我们从XML 配置文件的方式剖析了AOP的源码，我们也说了，虽然现在用XML配置的几乎没有了，但作为学习的例子，XML配置仍然是我们理解Spring AOP 的一个绝好的样本，但作为一个由追求的程序员，我们天天使用的注解方式的AOP 肯定也是要去看看到底是如何实现的。现在有了之前阅读 XML 配置的源码的基础，今天我们来阅读注解方式的源码也变得轻松起来。

还记得我们之前说过，XML 配置的AOP是使用 ProxyFactoryBean ，实现了 FactoryBean的接口，而FactoryBean是Spring特意留给开发者们扩展的接口，而Spring 留给开发者们不止一个扩展接口，比如 BeanPostProcess 接口，实现着接口就可以在每个Bean的生成前后做一些增强或自定义（具体Spring 留给我们有哪些扩展接口，楼主有机会将会再写一篇文章解析）。

接下来就要好好讲讲我们上篇文章漏讲的接口设计。这是我们理解 AOP 的基础。

先从 ProxyFactoryBean 讲起，这个熟悉的哥们。

## 1. ProxyFactoryBean 类结构

![](http://upload-images.jianshu.io/upload_images/4236553-8f7b7e8cbc75bd8b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.1 ProxyCreatorSupport  类结构图

它继承了 ProxyCreatorSupport 这个类，这个类很重要，我们看看该类结构

![](http://upload-images.jianshu.io/upload_images/4236553-e701dd5d103ed2b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类有2个重要的方法，分别是获取代理工厂，创建代理。那么代理工厂是什么呢？AopProxyFactory ，是个接口，只定义了一个方法：AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException， 并且该接口目前只有一个默认实现类：DefaultAopProxyFactory，该类主要重写了接口方法 createAopProxy， 内部逻辑是根据给定的类配置来创建不同的代理 AopProxy，那么 AopProxy 是什么呢？就是真正创建代理的工厂，他是一个接口，有3个实现：

### 1.2  AopProxy  接口

![](http://upload-images.jianshu.io/upload_images/4236553-bfe7da9a3822fafd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.3 AopProxy  继承结构

![](http://upload-images.jianshu.io/upload_images/4236553-9cc1dfc406fe22bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从名字上可以看出来，一个是 JDK 动态代理，一个是 Cglib 代理，ObjenesisCglibAopProxy 扩展了它的父类 CglibAopProxy，在 DefaultAopProxyFactory 的实现里，使用的就是 ObjenesisCglibAopProxy 来实现 Cglib 的代理。他们分别实现了自己的getProxy 方法用以创建代理。我们看看这两个类的继承结构：

### 1.4 JdkDynamicAopProxy 继承结构

![](http://upload-images.jianshu.io/upload_images/4236553-b3be043ca9c4dd64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1.5 CglibAopProxy 继承结构没什么，主要是众多内部类

![](http://upload-images.jianshu.io/upload_images/4236553-24378f87337615ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到哟非常多的内部类，这些内部类是为了实现了 Cglib 的各种回调而实现的。主要实现了 MethodInterceptor 接口，
Callback 接口，Joinpoint 接口，Invocation 接口等待，总之是实现了Spring 的 cglib 模块的各种接口。

说了那么多，我们回来看看 ProxyCreatorSupport  ，下面是 ProxyCreatorSupport  的继承结构

## 2. ProxyCreatorSupport   类继承图

![](http://upload-images.jianshu.io/upload_images/4236553-46b4346d53d57b3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类有3个子类， 其中就有一个我们熟悉的 ProxyFactoryBean，该类实现了我们熟悉的 FactoryBean，还有一个可以获取容器内Bean的 BeanFactoyAware 接口，第二个是陌生的 AspectJProxyFactory 类，该类是用于集成 Spring 和 AspectJ 的。而最后一个类ProxyFactory 就是我们今天的主角，Spring 的类注释说：`用于编程使用的AOP代理，而不是在bean工厂中通过声明式设置。这个类提供了一种简单的方法，可以在定制的用户代码中获取和配置AOP代理实例`，大概意思就是通过编程的方式获取Bean 代理吧，而不是通过配置文件的方式。我们今天就可以见识到。


## 3. AnnotationAwareAspectJAutoProxyCreator 类

这个类的名字很长，为什么要说这个类呢？还记得我们刚开始说的 BeanPostProcessor 扩展接口吗？我们说该接口是spring 留给开发人员自定义增强bean的接口。而该类则实现了该接口，看名字也知道，该类是根据注解自动创建代理的创建者类。我们看看他的类图：

![](http://upload-images.jianshu.io/upload_images/4236553-93afd4512e1ce792.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，最底层的该类实现了 BeanPostProcessor 接口，可以在每个bena生成前后做操作。该类由 Rod Johnson 编写，注释上是这么说的：`任何AspectJ注释的类都将自动被识别，它们也会被识别`。和我们预想的一致。

我们知道了 AnnotationAwareAspectJAutoProxyCreator 是根据注解自动创建代理，而该类也算是 ProxyBean 的代理，那么，和它一样继承抽象父类的其他几个类的作用是什么呢？来都来了，就看看吧！我们看看类图：

![](http://upload-images.jianshu.io/upload_images/4236553-a0bdae69da37cb7a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到该类由4个实现：他们实现了不同的创建代理的方式：
1. 匹配Bean的名称自动创建匹配到的Bean的代理，实现类BeanNameAutoProxyCreator
2. 根据Bean中的AspectJ注解自动创建代理，实现类AnnotationAwareAspectJAutoProxyCreator，也就是我们今天说的注解类。
3. 根据Advisor的匹配机制自动创建代理，会对容器中所有的Advisor进行扫描，自动将这些切面应用到匹配的Bean中，实现类DefaultAdvisorAutoProxyCreator
4. InfrastructureAdvisorAutoProxyCreator，该类只在 AopConfigUtils 中的静态块用到，该类的注释：自动代理创建者只考虑基础设施顾问bean，忽略任何应用程序定义的顾问。意思应该是只是Sprnig的基础代理，开发者的应用会忽略。有知道的同学可以告诉我。

加上我们的ProxyFactoryBean，一共5种实现方法。从这里可以看出Spring 对于扩展的软件设计是多么优秀。


那么我们就来看看 AnnotationAwareAspectJAutoProxyCreator 是如何创建代理的。我们说 BeanPostProcessor 是Spring 留给我们扩展的接口，那么他是如何定义的呢？我们看看该接口：

```java
public interface BeanPostProcessor {
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```
该接口定义了两个方法，一个是在bean初始化之前执行，一个是在bean初始化之后执行。也就是说，开发者可以在这两个方法中做一些有趣的事情。我们看看 AnnotationAwareAspectJAutoProxyCreator 是如何实现该方法的。实际上 AnnotationAwareAspectJAutoProxyCreator 的抽象父类已经实现了该方法，我们看看是如何实现的：

```java
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

	/**
	 * Create a proxy with the configured interceptors if the bean is
	 * identified as one to proxy by the subclass.
	 * @see #getAdvicesAndAdvisorsForBean
	 */
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}

```
before 方法直接返回了bean，并没有做什么增强操作，重点在after方法，我们可以看该方法的注释：如果bean是由子类标识的，那么就创建一个配置的拦截器的代理。Spring 就是在这里创建了代理，我们进入关键方法 wrapIfNecessary 看看。该方法用来包装给定的Bean。

```java
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```
我们已将可以看到一个关键一行代码：Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean))，创建一个代理，该方法携带了bean的Class对象，benaName， 通知器数组，还有一个包装过的单例Bean，我们看看该方法实现（该方法对于我们来说已经到终点了），

```java
	protected Object createProxy(
			Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}
```
我们能够看到其中有一行让我们激动的代码，就是 ProxyFactory proxyFactory = new ProxyFactory()，创建了一个ProxyFactory 对象，我们刚刚说过，该对象和ProxyFactoryBean一样，继承了 ProxyCreatorSupport， 因此也就和 ProxyFactoryBean 一样拥有了 getProxy 方法，到这里，我们的一切都豁然开朗。当然上面几行代码是设置了通知器链。我们先按下不表。

> ProxyFactoryBean 扩展了 FactoryBean 接口， AnnotationAwareAspectJAutoProxyCreator 扩展了 BenaPostProcessor 了接口，其目的都是在Bean生成的时候做增强操作，Spring 通过这两种方式，完成了两种不同的代理生成方式，但最终都是继承了 ProxyCreatorSupport 类，该类才是生成代理的核心类。

我们可以看看XML 配置方式和 注解方式的方法堆栈调用图，从中我们可以看出一些端倪：


**XML 配置方式堆栈图**

![](http://upload-images.jianshu.io/upload_images/4236553-a871bbe51757ee51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**注解配置方法堆栈图**

![](http://upload-images.jianshu.io/upload_images/4236553-dcb8067d88ae0e46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们可以看到两者在AbstractBeanFactory 的 doGetBean 方法开始分道扬镳，走向了不同的逻辑，那么我们看看到底哪里不同，直接看代码：我们看XML配置堆栈，在258行：

![](http://upload-images.jianshu.io/upload_images/4236553-6ebd125978fb6d76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从这里进入，该方法做了些什么呢？为什么让他们走向不同的路线？我们看看该方法：

![](http://upload-images.jianshu.io/upload_images/4236553-c9a53c9c01e81754.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法有个重要的判断：是否是 FactoryBean 的子类型，很明显，我们的XML配置的ProxyFacoroyBean 返回 false，而注解方式的Bean则返回 false，ProxyFacoroyBean 会一直向下走，直到创建代理，而注解方式则会直接返回。走到302行：

![](http://upload-images.jianshu.io/upload_images/4236553-44e7e4e5ccc84ecf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注解方式会执行 getSingleton 方法，最后触发 createBean 回调方法，完成创建代理的过程。

到此为止，现在我们知道了到 XML 配置方式的 AOP 和注解的方式AOP 的生成区别。我们可以开始总结了。

## 4. 总结

首先，通过分析源码我们知道注解方式和 XML 配置方式的底层实现都是一样的，都是通过继承 ProxyCreatorSupport 来实现的，不同的通过扩展不同的 Spring 提供的接口，XML 扩展的是FactoryBean 接口， 而注解方式扩展的是 BenaPostProcessor 接口，通过Spring 的扩展接口，能够对特定的Bean进行增强。而 AOP 正式通过这种方式实现的。这也提醒了我们，我们也可以通过扩展 Spring  的某些接口来增强我们需要的 Bean 的某些功能。当然，篇幅有限，我们这篇文章只是了解了XML 配置方式和注解方式创建代理的区别，关于如何 @Aspect 和 @Around 的底层实现，还有通知器的底层实现，我们还没有分析，但我们隐隐的感觉到，其实万变不离其宗，底层的也是通过扩展 advice 和 pointcut  接口来实现的。 我们将会在后面的文章继续分析 AOP 是如何编织通知的。


good luck ！！！！






















