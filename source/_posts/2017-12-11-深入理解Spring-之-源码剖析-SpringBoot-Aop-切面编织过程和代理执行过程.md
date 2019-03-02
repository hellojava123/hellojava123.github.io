---
layout: post
title: 深入理解Spring-之-源码剖析-SpringBoot-Aop-切面编织过程和代理执行过程
date: 2017-12-11 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-c4b003d0a9df69ba.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 源码：
![](http://upload-images.jianshu.io/upload_images/4236553-811b1a3202d0b159.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/4236553-d6e5c7b8778a1e16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/4236553-ed80419b292c033f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




## # 前言

前两篇文章我们分析了AOP的原理，知道了AOP的核心就是代理，通知器，切点。我们也知道了XML配置方式和注解方式的底层实现都是相同的，都是通过 ProxyCreatorSupport  创建代理，但是XML是通过 ProxyFactoryBean 来实现的，而 注解是通过 BeanPostProcessor 来实现创建代理的逻辑。

我们上篇讲解注解方式的文章，剖析了 AnnotationAwareAspectJAutoProxyCreator 这个类，这个类就是根据注解创建代理的默认类。那么到底该类在Spring 中是如何运行的？基于注解的 AOP 在现在流行的 SpringBoot 中是如何设计实现的？今天楼主就要以一个简单的demo来 debug 一次源码，彻底了解 SpringBoot 下的AOP 是如何运作的。

我们将分为几个方面来讲：
1. AnnotationAwareAspectJAutoProxyCreator  后置处理器注册过程。
2. 标注 @Aspect 注解的 bean 的后置处理器的处理过程
3. 创建代理的过程
4. 目标对象方法调用的过程

## 1. AnnotationAwareAspectJAutoProxyCreator  后置处理器注册过程

### 1.1 后置处理器继承图谱

我们先回顾一下该类的继承图谱 ，看图中红框部分，可以看到该类间接实现了 BeanPostProcessor 接口，而且也实现了 InstantiationAwareBeanPostProcessor 接口，InstantiationAwareBeanPostProcessor 接口继承了BeanPostProcessor 接口。

![](http://upload-images.jianshu.io/upload_images/4236553-d4c9a1ccb7919d3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们重点看看这两个接口的关系，InstantiationAwareBeanPostProcessor 接口通过继承 BeanPostProcessor 拥有了父接口的两个方法，父类的两个方法是在bean初始化前后做一些控制，而 InstantiationAwareBeanPostProcessor  自己增加的方法则是在bean的实例化做一些控制，顺序：先执行 InstantiationAwareBeanPostProcessor 自身的两个实例化方法，再执行父接口的初始化方法。

![](http://upload-images.jianshu.io/upload_images/4236553-e586c1039d657127.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们看到这些方法肯定会问，BeanPostProcessor 这个接口不是在初始化前和初始化后都能做一些操作吗，为什么只叫后置处理器，应该叫前后置处理器啊？那么我们就去看看源码，该方法被多处重写，我们查看重写该方法的所有的实现，可以看到所有处理该方法的逻辑都是直接返回bean，没有任何逻辑操作，可见该方法就是一个空方法。而 postProcessAfterInitialization 后置处理初始化方法各个实现类都有不同的操作。还有一个需要注意的的地方就是，InstantiationAwareBeanPostProcessor 和 BeanPostProcessor 这两个方法名称极其相似，注意区分，BeanPostProcessor 是初始化，InstantiationAwareBeanPostProcessor  是实例化，实例化先执行，初始化后执行。


为什么大张旗鼓的说后置处理器这个事情呢？因为 AnnotationAwareAspectJAutoProxyCreator 就是后置处理器啊，他继承了 AbstractAutoProxyCreator 抽象类，该类实现了后置处理器接口的方法。

既然是个后置处理器，那么就需要注册，我们就看看注册后置处理器的过程。

### 1.2 后置处理器注册过程

我们会议一下Spring 容器初始化的方法 refresh ，当时我们说，该方法是整个spring 的核心代码，所有的逻辑都是从这里作为入口，那么该方法中有注册后置处理器的逻辑吗？

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 为刷新准备应用上下文
            prepareRefresh();
			// 告诉子类刷新内部bean工厂，即在子类中启动refreshBeanFactory()的地方----创建bean工厂，根据配置文件生成bean定义
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// 在这个上下文中使用bean工厂
			prepareBeanFactory(beanFactory);
			try {
				// 设置BeanFactory的后置处理器// 默认什么都不做
				postProcessBeanFactory(beanFactory);
				// 调用BeanFactory的后处理器，这些后处理器是在Bean定义中向容器注册的
				invokeBeanFactoryPostProcessors(beanFactory);
				// 注册Bean的后处理器，在Bean创建过程中调用
				registerBeanPostProcessors(beanFactory);
				//对上下文的消息源进行初始化
				initMessageSource();
				// 初始化上下文中的事件机制
				initApplicationEventMulticaster();
				// 初始化其他的特殊Bean
				onRefresh();
				// 检查监听Bean并且将这些Bean向容器注册
				registerListeners();
				// 实例化所有的（non-lazy-init）单件
				finishBeanFactoryInitialization(beanFactory);
				//  发布容器事件，结束refresh过程
				finishRefresh();
			} catch (BeansException ex) {
				// 为防止bean资源占用，在异常处理中，销毁已经在前面过程中生成的单件bean
				destroyBeans();
				// 重置“active”标志
				cancelRefresh(ex);
				// Propagate exception to caller.
				throw ex;
			} finally {
				resetCommonCaches();
			}
		}
	}
```


我们看一下代码，可以看到，其中有一行代码，registerBeanPostProcessors(beanFactory)，注册后置处理器，携带BeanFactory参数，该方法会调用 PostProcessorRegistrationDelegate 的一个静态方法：

![](http://upload-images.jianshu.io/upload_images/4236553-d7f55959d7fbf004.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该静态方法很长，主要逻辑就是，从BeanFactory 中找到所有实现了 BeanPostProcessor 接口的bean，然后添加进集合，最后调用自身的 registerBeanPostProcessors 静态方法，循环调用 beanFactory 的addBeanPostProcessor 方法，将后置处理器添加进bean工厂。

![](http://upload-images.jianshu.io/upload_images/4236553-81866e2e2cae5b12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

bean工厂的 addBeanPostProcessor 方法如下：

![](http://upload-images.jianshu.io/upload_images/4236553-79144b5798143a93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将后置处理器放在一个集合中，并保证只有一个不会重复。然后判断是否 InstantiationAwareBeanPostProcessor 类型的后置处理器，如果有，就将状态hasInstantiationAwareBeanPostProcessors 改为 true。

这个时候也就是完成了后置处理器的注册过程。

## 2. 标注 @Aspect 注解的 bean 的后置处理器的处理过程

我们在使用 @Aspec 同时也会使用类似 @Component 的注解，表示这是一个Bean，只要是bean，就会调用getBean方法，只要调用 getBean 方法，都会调用 AbstractAutowireCapableBeanFactory 的 createBean 方法，该方法其中有一行函数调用：

![](http://upload-images.jianshu.io/upload_images/4236553-918a38a8f60e6094.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注释说，会返回一个代理，我们看看该方法到底是什么逻辑：

![](http://upload-images.jianshu.io/upload_images/4236553-35bae00d974ffcfb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们注意看该方法的 if 判断，hasInstantiationAwareBeanPostProcessors， 就是判断我们刚刚设置的状态，向下走，获取到Bean的类型，先调用 applyBeanPostProcessorsBeforeInstantiation 方法，如果该方法有返回值，则调用 applyBeanPostProcessorsAfterInitialization 方法，注意，上面调用的是  “实例化前方法”，下面那个调用的是 “初始化后方法”，是不同的接口实现。通常在第一次调用的的时候，是不会有返回值的。我们看看该方法实现：

![](http://upload-images.jianshu.io/upload_images/4236553-591acd118d547219.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


该方法会获取所有的后置处理器，就是之前注册在BeanFactory 的后置处理器，并循环调用他们的 postProcessBeforeInstantiation 方法，但是还是要判断是不是 InstantiationAwareBeanPostProcessor 类型，我们关注的当然是我们在之前的说的 AbstractAutoProxyCreator 后置处理器，我们看看他的 postProcessBeforeInstantiation 方法：

![](http://upload-images.jianshu.io/upload_images/4236553-4eb22a60fbd9b84a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法首先检验参数，如果缓存中存在的话就直接返回。如果不在，则判断该bean的类型是否是基础类型，上面是基础类型？ 我们看代码：

![](http://upload-images.jianshu.io/upload_images/4236553-1f9a10f5fc032543.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


该方法首先调用父类的的 isInfrastructureClass 方法，然后调用 isAspcet 方法。我们看看这两个方法的实现：

**isInfrastructureClass**

![](http://upload-images.jianshu.io/upload_images/4236553-f33bd77d62f2dd56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

判断是否是这几个基础类。

**isAspect**

![](http://upload-images.jianshu.io/upload_images/4236553-246c3ee1e44c9ede.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

判断该类是否含有Aspect 注解并且该类属性中含有 “ajc$” 开头的成员变量。

那么后面的那个判断呢？shouldSkip ，字面意思：是否应该跳过。  该方法默认实现是false，但该类注释说，子类需要重写该方法。我们看看该方法是如何重写的：

![](http://upload-images.jianshu.io/upload_images/4236553-22b53568dfeb4d04.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


首先找到所有候选的通知器，然后循环通知器数组，如果该通知器是 AspectJPointcutAdvisor 类型，并且该通知器的通知的切面名称和bean的名字相同，就返回 true。重点在如果获取通知器，在XML配置中，我们知道，通知器是我们显式的实现了 PointcutAdvisor 接口的类，并在配置文件中配置。而这里我们没有配置。Spring 是如何找到的呢？我们看看 findCandidateAdvisors 方法，该方法委托了 advisorRetrievalHelper 调用 findAdvisorBeans 方法，我们看看该方法实现。
注意：findCandidateAdvisors 方法被重写了，我们看子类实现。

![](http://upload-images.jianshu.io/upload_images/4236553-7fca130fe119060f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

子类方法比父类方法多了一行 buildAspectJAdvisors。

```java
public List<Advisor> findAdvisorBeans() {
		// Determine list of advisor bean names, if not cached already.
		String[] advisorNames = null;
		synchronized (this) {
			advisorNames = this.cachedAdvisorBeanNames;
			if (advisorNames == null) {
				// Do not initialize FactoryBeans here: We need to leave all regular beans
				// uninitialized to let the auto-proxy creator apply to them!
				advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
						this.beanFactory, Advisor.class, true, false);
				this.cachedAdvisorBeanNames = advisorNames;
			}
		}
		if (advisorNames.length == 0) {
			return new LinkedList<Advisor>();
		}

		List<Advisor> advisors = new LinkedList<Advisor>();
		for (String name : advisorNames) {
			if (isEligibleBean(name)) {
				if (this.beanFactory.isCurrentlyInCreation(name)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipping currently created advisor '" + name + "'");
					}
				}
				else {
					try {
						advisors.add(this.beanFactory.getBean(name, Advisor.class));
					}
					catch (BeanCreationException ex) {
						Throwable rootCause = ex.getMostSpecificCause();
						if (rootCause instanceof BeanCurrentlyInCreationException) {
							BeanCreationException bce = (BeanCreationException) rootCause;
							if (this.beanFactory.isCurrentlyInCreation(bce.getBeanName())) {
								if (logger.isDebugEnabled()) {
									logger.debug("Skipping advisor '" + name +
											"' with dependency on currently created bean: " + ex.getMessage());
								}
								// Ignore: indicates a reference back to the bean we're trying to advise.
								// We want to find advisors other than the currently created bean itself.
								continue;
							}
						}
						throw ex;
					}
				}
			}
		}
		return advisors;
	}

```
该方法会通过  BeanFactoryUtils.beanNamesForTypeIncludingAncestors 方法获取容器中所有实现了Advisor 接口的Bean，然而我们的demo程序里上面都没有。如果是XML就会返回了。

那我们看第二行的 this.aspectJAdvisorsBuilder.buildAspectJAdvisors(）方法，该方法根据注解创建通知器。我们看看该方法逻辑：

```java
	public List<Advisor> buildAspectJAdvisors() {
		List<String> aspectNames = this.aspectBeanNames;

		if (aspectNames == null) {
			synchronized (this) {
				aspectNames = this.aspectBeanNames;
				if (aspectNames == null) {
					List<Advisor> advisors = new LinkedList<Advisor>();
					aspectNames = new LinkedList<String>();
					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
					for (String beanName : beanNames) {
						if (!isEligibleBean(beanName)) {
							continue;
						}
						// We must be careful not to instantiate beans eagerly as in this case they
						// would be cached by the Spring container but would not have been weaved.
						Class<?> beanType = this.beanFactory.getType(beanName);
						if (beanType == null) {
							continue;
						}
						if (this.advisorFactory.isAspect(beanType)) {
							aspectNames.add(beanName);
							AspectMetadata amd = new AspectMetadata(beanType, beanName);
							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
								MetadataAwareAspectInstanceFactory factory =
										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
								if (this.beanFactory.isSingleton(beanName)) {
									this.advisorsCache.put(beanName, classAdvisors);
								}
								else {
									this.aspectFactoryCache.put(beanName, factory);
								}
								advisors.addAll(classAdvisors);
							}
							else {
								// Per target or per this.
								if (this.beanFactory.isSingleton(beanName)) {
									throw new IllegalArgumentException("Bean with name '" + beanName +
											"' is a singleton, but aspect instantiation model is not singleton");
								}
								MetadataAwareAspectInstanceFactory factory =
										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
								this.aspectFactoryCache.put(beanName, factory);
								advisors.addAll(this.advisorFactory.getAdvisors(factory));
							}
						}
					}
					this.aspectBeanNames = aspectNames;
					return advisors;
				}
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
		List<Advisor> advisors = new LinkedList<Advisor>();
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}
```

方法很长，我们看看主要逻辑：先调用静态方法 BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Object.class, true, false) ，也就是获取容器中所有的bean，然后循环处理每个bean。这个 buildAspectJAdvisors 有个注意的地方就是，每次执行该方法，最后都会更新 aspectBeanNames 属性，该属性是通知器的名称集合。在循环中，会先创建一个 AspectMetadata 切面元数据对象，然后创建 BeanFactoryAspectInstanceFactory 对象，在使用 advisorFactory 的 getAdvisors 方法携带刚刚创建的 factory 对象获取该Class 对应的所有 通知器。

我们看看 getAdvisors 的标准实现：

```java
	@Override
	public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new LinkedList<Advisor>();
		for (Method method : getAdvisorMethods(aspectClass)) {
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}
```

在 getAdvisors 方法中，会先根据Bean 的Class，比如 Aop.class，和 切面名称，也就是benaNanme，然后根据class，获取所有不带 @Pointcut.class 的 Method 对象集合，也就是 Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class 这些注解。拿到这些方法数组后，进行循环处理，调用自己的 getAdvisor 方法，我们看看 getAdvisor 方法。

```java
	@Override
	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) {

		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}

		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}
```

在该方法中，则根据method 对象获取Pointcut，getPointcut 会将该方法上面的注解解析成一个 AspectJExpressionPointcut 切面表达式切入点对象，最后 getAdvisor 将所有参数和 刚刚创建的对象 包装成 InstantiationModelAwarePointcutAdvisorImpl 对象也就是 Advisor 的子类 返回。

回到 getAdvisors 方法中， 该方法会将 getAdvisor 方法的返回值添加到 Advisors 数组中，然后如果数组不为空的话，创建一个合成的，没有作用的通知器。 然后，循环切面类的所有属性，解析含有 DeclareParents 注解的成员变量。

至此，得到了该类所有的通知器。然后放入缓存。最后 ，将 aspectBeanNames 属性重新赋值。并返回 advisors。

回到 findCandidateAdvisors 方法，也就是寻找候选的通知器。然后将创建的通知器返回。

回到  AbstractAutoProxyCreator 的 postProcessBeforeInstantiation 的方法， 该方法的 shouldSkip 判断返回false， 没能进入if块返回，继续向下走。 

首先寻找 Bean 的目标类，我们并没有设置目标类，然后，就返回null。至此，resolveBeforeInstantiation 方法返回了null。并没有像注释说的返回一个代理，除非设置了 customTargetSourceCreators。虽然这里也能创建代理。而该方法也就结束了。该方法主要执行了后置处理器的 “实例化之前” 方法。


## 3. 创建代理的过程

在执行完 resolveBeforeInstantiation 返回 null 之后，就执行 doCreateBean 方法，我们很熟悉这个方法，该方法在调用 populateBean 方法后完成了依赖注入，然后进行初始化， 调用 initializeBean 方法。

![](http://upload-images.jianshu.io/upload_images/4236553-c75d6790082ba381.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**initializeBean 方法**

![](http://upload-images.jianshu.io/upload_images/4236553-17699c53bcffcae8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

initializeBean  方法中调用了3个重要的逻辑，applyBeanPostProcessorsBeforeInitialization 方法 ， 和 invokeInitMethods 方法（如果实现了InitializingBean 接口，则会调用 afterPropertiesSet 方法），还有重要的 applyBeanPostProcessorsAfterInitialization 方法。

首先调用了前置处理器的 postProcessBeforeInitialization 方法，简单的返回了bean。
然后判断是否实现 InitializingBean  接口从而决定是否调用  afterPropertiesSet  方法。

 最后到了最重要的 applyBeanPostProcessorsAfterInitialization  方法， 该方法会循环调用每个bean的后置处理器的 postProcessAfterInitialization 方法。

![](http://upload-images.jianshu.io/upload_images/4236553-f909b6dc9103de4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在，来到了我们熟悉的 postProcessAfterInitialization 方法，该方法重写了 BeanPostProcessor 接口的方法，然后进入到 wrapIfNecessary 方法。

![](http://upload-images.jianshu.io/upload_images/4236553-80c3acf11b904dc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先判断是否 targetSourcedBeans 目标bean 集合中是否存在，有则直接返回。
再判断对应的bean是否含有通知器。
再判断是不是基础类，我们之前看过了，如果有，则返回true。也就会直接返回bean。或则调用 shouldSkip 方法判断，如果其中某个通知器 实现了AspectJPointcutAdvisor 接口，则返回 true。我们的Bean 两者都不是，所以都是false，向下走。

这里就是我们曾经分析过的代码，今天再分析一下，主要还是创建代理。首先获取所有的通知器。如果在后置处理器注册的时候该Bean没有设置通知器，则不会创建代理。然后还有就是我们之前注册的后置处理器了。如过后置处理不是null，则再缓存中存储bean的名字和是否有通知器的boolean值，再创建完代理后，将代理类和bean的名字存入代理类型缓存。以便以后可以随时实例化代理。

![](http://upload-images.jianshu.io/upload_images/4236553-093d9f04424e9f0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

createProxy 是我们的老朋友，首先判断 BeanFactory 是否实现了 ConfigurableListableBeanFactory 接口，默认是实现了的。调用 AutoProxyUtils.exposeTargetClass 方法，

然后创建 ProxyFactory 实例，这个我们分析过了，然后将 AnnotationAwareAspectJAutoProxyCreator 数据复制到 ProxyFactory 中。


![](http://upload-images.jianshu.io/upload_images/4236553-11c6733d7d15a050.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


buildAdvisors  创建通知器数组，根据bean的名称，和刚刚注册的通知器。我们看看该方法，该方法首先调用 resolveInterceptorNames 方法，生成通知器数组，但我们没有设置该属性（XML 配置曾经设置过）。 然后将我们给定的通知器添加到一个数组，然后判断，如果配置了通知器，则将配置的通知器放在数组前面， 否则就放在后面。

最后，循环该通知器数组，调用 advisorAdapterRegistry 的warp 方法，包装通知器。最后返回，如何包装呢？

![](http://upload-images.jianshu.io/upload_images/4236553-7223ae4f79bae998.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看看warp 方法，判断如果是Advisor 类型，则直接返回，如果不是Advice类型，抛出异常，如果是，则判断是否是 MethodInterceptor 类型，如果是，则包装成 DefaultPointcutAdvisor 类型返回。如果都不满足，则循环该类的adappters 适配器数组， 判断如果支持 Advice ，则也创建一个默认的 DefaultPointcutAdvisor 返回。该适配器数组，默认就有3个适配器，一个before，一个after，一个throws。而我们的通知器都是 Advisor，所以直接返回了。

到这里 buildAdvisors 方法也就结束了，返回了一个Advisors 数组。

回到 createProxy 方法， 得到了注册的通知器，于是 proxyFactory 可以设置 通知器属性了，也设置了目标类属性，这时候就可以拿着工厂去创建代理了。

如何创建代理工厂我们就不看了，因为我们之前已经分析过了，我们的demo肯定创建的是 Cglib 代理，因此我们进入到Cglib 的getProxy 方法中查看。

```java
	@Override
	public Object getProxy(ClassLoader classLoader) {
		try {
			Class<?> rootClass = this.advised.getTargetClass();
			Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

			Class<?> proxySuperClass = rootClass;
			if (ClassUtils.isCglibProxyClass(rootClass)) {
				proxySuperClass = rootClass.getSuperclass();
				Class<?>[] additionalInterfaces = rootClass.getInterfaces();
				for (Class<?> additionalInterface : additionalInterfaces) {
					this.advised.addInterface(additionalInterface);
				}
			}

			// Validate the class, writing log messages as necessary.
			validateClassIfNecessary(proxySuperClass, classLoader);

			// Configure CGLIB Enhancer...
			Enhancer enhancer = createEnhancer();
			if (classLoader != null) {
				enhancer.setClassLoader(classLoader);
				if (classLoader instanceof SmartClassLoader &&
						((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
					enhancer.setUseCache(false);
				}
			}
			enhancer.setSuperclass(proxySuperClass);
			enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

			Callback[] callbacks = getCallbacks(rootClass);
			Class<?>[] types = new Class<?>[callbacks.length];
			for (int x = 0; x < types.length; x++) {
				types[x] = callbacks[x].getClass();
			}
			// fixedInterceptorMap only populated at this point, after getCallbacks call above
			enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
			enhancer.setCallbackTypes(types);

			// Generate the proxy class and create a proxy instance.
			return createProxyClassAndInstance(enhancer, callbacks);
		}
	}
```

首先判断是否是Cglib类型，怎么判断呢？就判断类名中是否含有两个$$ 符号，如果是的，则从当前类中获取所有接口，并添加到 advised 属性中，很明显，我们不是。

![](http://upload-images.jianshu.io/upload_images/4236553-08f11ff29f49e2a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4236553-d5e8b8e2f0a1ffac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后校验Class，写日志。

创建一个增强器。是 org.springframework.cglib.proxy 包下的。给增强器设置类加载器，设置父类，设置接口，设置命名规则，默认是 BySpringCGLIB，设置策略（不知道是什么意思），，设置回调类型（七个回调，都是CglibAopProxy 的内部类），设置了回调的过滤器，最后创建代理，再下面我们就不深究了。

我们从整个流程可以看到，AOP的编织是通过定义@Aspect 和 @Around 等注解创建通知器，和我们在XML中编程几乎一样，只是Sprng 向我们屏蔽了底层。最后，Spring 拿着目标类和通过器去创建代理。


## 4. 目标对象方法调用的过程

那么创建代理后做什么呢？

这时候就需要调用被 @PostConstruct 注解 的方法了，该注解的作用就是在Spring 实例化该bena后执行该方法。

![](http://upload-images.jianshu.io/upload_images/4236553-3e914591cc018fbb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时返回的已经是代理类了，代理类执行testController 方法，代理类内部回调了 CglibAopProxy 的内部类 DynamicAdvisedInterceptor 的 intercept 方法，

```java
		@Override
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Class<?> targetClass = null;
			Object target = null;
			try {
				if (this.advised.exposeProxy) {
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				target = getTarget();
				if (target != null) {
					targetClass = target.getClass();
				}
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null) {
					releaseTarget(target);
				}
				if (setProxyContext) {
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}

```
首先获取目标类， 然后传入目标方法和 目标类获取通知器链， 最终调用 DefaultAdvisorChainFactory 的 getInterceptorsAndDynamicInterceptionAdvice 方法， 该方法参数为  AdvisedSupport， 目标方法，目标类。我们进入该方法查看：

```java
	@Override
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, Class<?> targetClass) {

		// This is somewhat tricky... We have to process introductions first,
		// but we need to preserve order in the ultimate list.
		List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

		for (Advisor advisor : config.getAdvisors()) {
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
						if (mm.isRuntime()) {
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}
```

首先调用 hasMatchingIntroductions 方法，返回false，该方法会判断通知器是否是 IntroductionAdvisor 类型，如果是，则判断类过滤器是否匹配当前的目标类型。我们这里不是 IntroductionAdvisor 类型，因此都返回false。然后获取全局的三个通知器适配器，然后循环匹配，最后将匹配成功的包装到 InterceptorAndDynamicMethodMatcher 中后添加到拦截器数组中。

回到  intercept ,得到了拦截器数组，如果拦截器数组是空的并且方法修饰符是public 的，就直接调用该方法，如果不是，则创建一个Cglib 方法调用器。并执行proceed方法，

![](http://upload-images.jianshu.io/upload_images/4236553-8a29bc66cbefbbba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于我们的方法不是public 的，则执行 else 逻辑，我们看看该 proceed 方法。

```java
	@Override
	public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed(); 
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this); // 执行这里
		}
	}
````

proceed 方法会先获取第一个拦截器，如果拦截器不是 InterceptorAndDynamicMethodMatcher 类型的，则执行拦截器的invoke 方法。

![](http://upload-images.jianshu.io/upload_images/4236553-bac2d17a91f35eb3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在invoke 方法中，先将 mehodInvocation 对象放置在ThreadLocal 中，然后又回调自己的proceed方法，判断当前拦截器下标是否到了最大。再次执行的拦截器也就是通知器是 AspectJAroundAdvice 的invoke 方法，也就是我们定义的@Around 注解的方法。

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
		ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
		JoinPointMatch jpm = getJoinPointMatch(pmi);
		return invokeAdviceMethod(pjp, jpm, null, null);
	}
```


首先创建一个我们熟知的 ProceedingJoinPoint 对象，然后调用 invokeAdviceMethod 方法，是其抽象父类的AbstractAspectJAdvice 的方法。该方法继续调用invokeAdviceMethodWithGivenArgs 方法。

```java
	protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
		Object[] actualArgs = args;
		if (this.aspectJAdviceMethod.getParameterTypes().length == 0) {
			actualArgs = null;
		}
		try {
			ReflectionUtils.makeAccessible(this.aspectJAdviceMethod);
			// TODO AopUtils.invokeJoinpointUsingReflection
			return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);
		}
		catch (IllegalArgumentException ex) {
			throw new AopInvocationException("Mismatch on arguments to advice method [" +
					this.aspectJAdviceMethod + "]; pointcut expression [" +
					this.pointcut.getPointcutExpression() + "]", ex);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}

````

终于，在这里调用我们的test方法了。执行反射方法this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs） 。

![](http://upload-images.jianshu.io/upload_images/4236553-33165987eefb4864.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后回调自己的proceed 方法，回到了 ReflectiveMethodInvocation 的 proceed 方法，继续判断是否还有拦截器，没有则执行 invokeJoinpoint 方法，也就是目标类本身的方法。

![](http://upload-images.jianshu.io/upload_images/4236553-87c3aa4b21d8450f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/4236553-07b7b0f94d4f6832.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果方法是public 的，则直接执行方法代理的invoke方法，如果不是，执行父类的 invokeJoinpoint 方法， 实际上是执行的 AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments) 方法，该方法执行反射方法 method.invoke ，最终执行目标方法。

![](http://upload-images.jianshu.io/upload_images/4236553-031fc424f99d1ae6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4236553-310e620c947bc5ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最终返回到拦截器的方法中，也就是我们的Aop.tesst 方法，继续执行下面的逻辑。

执行完毕，AbstractAspectJAdvice 通知器的invokeAdviceMethod 的方法结束，开始逐层返回。返回到 intercept 方法， 继续向下执行。下面就是一些处理返回值的逻辑，最后返回返回值。完成了代理的一次调用。

![image.png](http://upload-images.jianshu.io/upload_images/4236553-9c4715d518893dcb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 5. 总结

这篇文章可以说是非常的长，从 AOP 的切面编织，到代理的生成，到方法的调用，可以说，非常的复杂。 不过，Spring 中最复杂的 AOP 我们已经彻底搞清楚了他的原理，功夫不负苦心人。不论是老旧的 XML 配置方式，还是新的 SpringBoot 方式，我们都能够了解他的原理，在以后的程序排错中，也更加游刃有余，实际上，我们得到的不止是这个，我们知道了Spring 的众多扩展接口，这些接口是AOP 赖以生存的条件，正式通过他们，完成了AOP的增强。我们也知道了，无论是注解还是配置文件，都是依赖同一个底层原理，这也警醒了我们，无论多么复杂的系统，底层都是相似的，通过层层封装后变得更加好用的同时也变得难以理解，但只要明白底层原理，一切都没那么复杂，楼主认为底层原理更是网络，OS，计组，编译原理，数据结构这些。山高路远，我们将一直走下去。


good luck !!!!!











