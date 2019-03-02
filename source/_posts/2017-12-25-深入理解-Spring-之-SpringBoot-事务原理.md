---
layout: post
title: 深入理解-Spring-之-SpringBoot-事务原理
date: 2017-12-25 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-9c798b014a065a06.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

今天是平安夜，先祝大家平安夜快乐。

我们之前的数十篇文章分析了 Spring 和 Mybatis 的原理，基本上从源码层面都了解了他们的基本原理，那么。在我们日常使用这些框架的时候，还有哪些疑问呢？就楼主而言，楼主已经明白了 IOC ，AOP 的原理，也明白了 Mybatis 的原理，也明白了 Spring 和 Mybatis 是如何整合的。但是，我们漏掉了 JavaEE 中一个非常重要的特性：事务。事务是 Java 程序员开发程序时不可避免的问题。我们就不讨论 ACID 的事务特性，楼主这里假定大家都已经了了解了事务的原理。如果还不了解，可以先去谷歌看看。那么，我们今天的任务是剖析源码，看看Spring 是怎么运行事务的，并且是基于当前最流行的SpringBoot。还有，我们之前剖析Mybatis 的时候，也知道，Mybatis 也有事务，那么，他俩融合之后，事务是交给谁的？又是怎么切换的？今天这几个问题，我们都要从源码中找到答案。

## 1. Spring 的事务如何运行？

如果各位使用过SpringBoot ，那么就一定知道如何在Spring中使用注解，比如在一个类或者一个方法上使用 @Transactional 注解，在一个配置类上加入一个 @EnableTransactionManagement 注解代表启动事务。而这个配置类需要实现 TransactionManagementConfigurer 事务管理器配置接口。并实现 annotationDrivenTransactionManager 方法返回一个包含了 配置好数据源的 DataSourceTransactionManager 事务对象。这样就完成了事务配置，就可以在Spring使用事务的回滚或者提交功能了。

![](http://upload-images.jianshu.io/upload_images/4236553-cfc259620f573921.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4236553-112ad1d23f28d020.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4236553-4900c9a43c325e49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个 saveList 方法就在Spring事务的控制之下，如果发生了异常，就会回滚事务。如果各位知道更多的Spring的事务特性，可以在注解中配置，比如什么异常才能回滚，比如超时时间，比如隔离级别，比如事务的传播。就更有利于理解今天的文章了。

我们基于一个 Junit 测试用例，来看看Spring的事务时如何运行的。

![](http://upload-images.jianshu.io/upload_images/4236553-60b40a98331487dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在测试用例中执行该方法，参数时一个空的List，这个Sql的运行肯定是失败的。我们主要看看他的运行过程。我们讲断点打在该方法上。断点进入该方法。

![](http://upload-images.jianshu.io/upload_images/4236553-603d4e012ceea008.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意，dataCollectionShareService 对象已经被 Cglib 代理了，那么他肯定会走 DynamicAdvisedInterceptor 的 intercept 方法，我们断点进入该方法查看，这个方法我们已经很属性了，该方法中，最重要的事情就是执行通知器或者拦截器的方法，那么，该代理有通知器吗？

![](http://upload-images.jianshu.io/upload_images/4236553-9a3f166c88b56ec3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有一个通知器。是什么呢？

![](http://upload-images.jianshu.io/upload_images/4236553-40d107782c659d0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一个事务拦截器，也就是说，如果通知器链不为空，就会依次执行通知器链的方法。那么 TransactionInterceptor 到底是什么呢？

![](http://upload-images.jianshu.io/upload_images/4236553-9a30b4722ed6458a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类实现了通知器接口，也实现类 MethodInterceptor 接口，并实现了该接口的 invoke 方法，在 DynamicAdvisedInterceptor 的 intercept 方法中，最终会调用每个 MethodInterceptor 的 invoke 方法，那么，TransactionInterceptor  的 invoke 方法是如何实现的呢？

![](http://upload-images.jianshu.io/upload_images/4236553-bdd281b4ec68bffb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

invoke 方法中会调用自身的 invokeWithinTransaction 方法，看名字，该方法和事务相关。该方法参数是由目标方法，目标类，一个回调对象构成。 那么我们就进入该方法查看，该方法很长：

```java
	/**
	 * General delegate for around-advice-based subclasses, delegating to several other template
	 * methods on this class. Able to handle {@link CallbackPreferringPlatformTransactionManager}
	 * as well as regular {@link PlatformTransactionManager} implementations.
	 * @param method the Method being invoked
	 * @param targetClass the target class that we're invoking the method on
	 * @param invocation the callback to use for proceeding with the target invocation
	 * @return the return value of the method, if any
	 * @throws Throwable propagated from the target invocation
	 */
	protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
			throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
						new TransactionCallback<Object>() {
							@Override
							public Object doInTransaction(TransactionStatus status) {
								TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
								try {
									return invocation.proceedWithInvocation();
								}
								catch (Throwable ex) {
									if (txAttr.rollbackOn(ex)) {
										// A RuntimeException: will lead to a rollback.
										if (ex instanceof RuntimeException) {
											throw (RuntimeException) ex;
										}
										else {
											throw new ThrowableHolderException(ex);
										}
									}
									else {
										// A normal return value: will lead to a commit.
										return new ThrowableHolder(ex);
									}
								}
								finally {
									cleanupTransactionInfo(txInfo);
								}
							}
						});

				// Check result: It might indicate a Throwable to rethrow.
				if (result instanceof ThrowableHolder) {
					throw ((ThrowableHolder) result).getThrowable();
				}
				else {
					return result;
				}
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
		}
	}
```

该方法主要逻辑：

1. 获取事务属性，根据事务属性，获取事务管理器。
2. 判断属性是否空，或者事务管理器是否不是 CallbackPreferringPlatformTransactionManager 类型，如果是该类型，则会执行事务管理器的 execute 方法。
3. 生成一个封装了事务管理器，事务属性，方法签名字符串，事务状态对象 的 TransactionInfo 事务信息对象。该对象会在事务回滚或者失败时起作用。
4. 调用目标对象方法或者是下一个过滤器的方法。
5. 如果方法由异常则执行 completeTransactionAfterThrowing 方法，调用事务管理器的回滚方法。如果没有异常，调用 commitTransactionAfterReturning 提交方法。最后返回返回值。

可以说，该方法就是Spring 事务的核心调用，根据目标方法是否有异常进行事务的回滚。

那么，我们需要一行一行的看看该方法实现。

首先看事务的属性。

## 2. TransactionAttribute 事务属性

invokeWithinTransaction 方法中调用了 自身的 getTransactionAttributeSource 方法返回一个TransactionAttributeSource 对象，并调用该对象的 getTransactionAttribute 方法，参数是目标方法和目标类对象。首先看 getTransactionAttributeSource 方法，该方法直接返回了抽象类 TransactionAspectSupport 中定义的 TransactionAttributeSource 属性。该属性的是什么时候生成的我们稍后再说。我们debug 后返回的是 TransactionAttributeSource 接口的实现类 AnnotationTransactionAttributeSource ，看名字，注解事务属性资源，名字起的好很重要啊。我们进入该类查看。

![](http://upload-images.jianshu.io/upload_images/4236553-bec4cff1542d000a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是该类的继承机构图。我们重点还是关注该类的 getTransactionAttribute 方法，该方法有抽象类 AbstractFallbackTransactionAttributeSource 也就是 AnnotationTransactionAttributeSource  的父类完成。我们看看该方法。

![](http://upload-images.jianshu.io/upload_images/4236553-8c063dce4193dae8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法大部分都是缓存判断，最重要的一行代码楼主已红框标出。computeTransactionAttribute 方法，计算事务属性。进入该方法查看：

![](http://upload-images.jianshu.io/upload_images/4236553-63eb53d0c640cd0c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法是返回事务属性的核心方法，首先，根据 class 和 method 对象，生成一个完整的method 对象，然后调用 findTransactionAttribute 方法，参数就是该 method 对象，findTransactionAttribute 方法是抽象方法，由子类实现，可见 computeTransactionAttribute 是个模板方法模式。那么我们就看看他的子类 AnnotationTransactionAttributeSource 是如何实现的。该方法调用了自身的 determineTransactionAttribute 方法。该方法实现入下：

![](http://upload-images.jianshu.io/upload_images/4236553-c747b5f262b7d2db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法会判断该 Method 对象是否含有注解。并循环 AnnotationTransactionAttributeSource 对象的 annotationParsers 注解解析器集合，对该方法进行解析。如果解析成功，则返回该注解元素。我想我们也已经猜到了，这个注解解析器解析的就是  @Transactional 注解。


## 3. @Transactional  注解解析器 SpringTransactionAnnotationParser

我们说AnnotationTransactionAttributeSource  对象中又多个解析器。那么这些解析器是什么时候生成的呢？构造方法中生成的。

![](http://upload-images.jianshu.io/upload_images/4236553-00711cd6d27a7276.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该构造方法由一个布尔属性，然后创建一个链表，也创建一个 SpringTransactionAnnotationParser 对象添加进链表中。这样就完成了解析器的创建。构造方法什么时候调用的呢？我们稍后再讲。

我们看看注解解析器是怎么解析方法对象的。

![](http://upload-images.jianshu.io/upload_images/4236553-96230369822a2ef5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先根据指定的 Transactional 注解和给定的方法，调用工具方法 getMergedAnnotationAttributes ，获取方法上的注解属性。然后调用重载方法 parseTransactionAnnotation 。

![](http://upload-images.jianshu.io/upload_images/4236553-d12844be8788d1e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，该方法首先创建了一个 RuleBasedTransactionAttribute 对象，然后一个个解析注解中的元素，并将这些元素设置到 RuleBasedTransactionAttribute 对象中，注意，其中有个 RollbackRuleAttribute 的集合，存储着该注解属性的回滚相关的属性。最后添加到 RuleBasedTransactionAttribute 的RollbackRules 集合中。

到这里，就完成了解析器的解析。返回了一个 RuleBasedTransactionAttribute 对象。

回到 拦截器的 invokeWithinTransaction  方法中，此时已经获取了 属性对象。根据方法，也就是说，如果返回值是null，说明该方法没有事务注解，在 getTransactionAttribute 方法中，也会将该方法作为key ，NULL_TRANSACTION_ATTRIBUTE 作为 value，放入缓存，如果不为null，那么就将 TransactionAttribute 作为 value 放入缓存。

有了事务属性，再获取事务管理器。也就是 determineTransactionManager 方法。

## 4. 事务管理器。

![](http://upload-images.jianshu.io/upload_images/4236553-5d51433cdd8fce48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们注意到，调用了自身的 determineTransactionManager 方法，返回了一个 PlatformTransactionManager 事务管理器。这个事务管理器就是我们在我们的配置类中写的：

![](http://upload-images.jianshu.io/upload_images/4236553-4ba143baaa43e87a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么这个事务管理器是什么呢？事务管理器就是真正执行事务回滚或提交的执行单位，我们看看该类：

继承图：
![](http://upload-images.jianshu.io/upload_images/4236553-7241daa34fa74cc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

结构图：
![](http://upload-images.jianshu.io/upload_images/4236553-4d9ecce214dbd360.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

红框标注的方法就是执行正在事务逻辑的方法，其中又封装了数据源，也就是 JDBC 的 Connection 。比如 doCommit 方法：

![](http://upload-images.jianshu.io/upload_images/4236553-e16bd2db1474426e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们看看determineTransactionManager 是如何获取事务管理器的。

![](http://upload-images.jianshu.io/upload_images/4236553-eaf7239f86432942.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法步骤入下：
1. 如果事务属性为null 或者 容器工厂为null，则返会自身的 transactionManager 事务管理器。
2. 如果都不为null，则获取事务属性的限定符号，根据限定符从容器中获取 事务管理器。
3. 如果没有限定符，则根据事务管理器的BeanName从容器中获取。
4. 如果都没有，则获取自身的事务管理器，如果自身还没有，则从缓存中取出默认的。如果默认的还没有，则从容器中获取PlatformTransactionManager 类型的事务管理器，最后返回。

这里重点是自身的事务管理器从何而来？我们先按下不表。

到这里，我们已经有了事务管理器。就需要执行 invokeWithinTransaction 下面的逻辑了。回到 invokeWithinTransaction 方法，我们的返回值肯定满足第一个if 条件，因为我们的事务管理器不是 CallbackPreferringPlatformTransactionManager 类型的。进入if 块。

![](http://upload-images.jianshu.io/upload_images/4236553-557f6911d9b99e69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先创建一个事务信息对象。该类是什么呢？

属性：
![](http://upload-images.jianshu.io/upload_images/4236553-e8a8083b7590c398.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

构造方法：
![](http://upload-images.jianshu.io/upload_images/4236553-a88c1a38834531eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类包含了一个 事务管理器，事务属性，事务方法字符串。

接着执行回调类InvocationCallback 的 proceedWithInvocation 方法，该方法会执行下一个通知器的拦截方法（如果有的话），最后执行目标方法，这里，目标方法被 try 住了，如果发生异常，则执行completeTransactionAfterThrowing 方法，并抛出异常，在 finally 块中执行清理工作。如果成功执行，则执行 
commitTransactionAfterReturning 方法。最后返回目标方法返回值。

我们重点看看 completeTransactionAfterThrowing 方法和 commitTransactionAfterReturning 方法。

## 5. TransactionInterceptor 的 completeTransactionAfterThrowing 方法（事务如何回滚）。

![](http://upload-images.jianshu.io/upload_images/4236553-5c15034853258152.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法主要内容在红框中，首先判断该事务对象是否和该异常匹配，如果匹配，则回滚，否则，则提交。那么，是否匹配的逻辑是怎么样的呢？我们的事务属性是什么类型的？RuleBasedTransactionAttribute ，就是我们刚刚创建解析注解后创建的。那么我就看看该类的 rollbackOn 方法：

![](http://upload-images.jianshu.io/upload_images/4236553-73df2cdd08589066.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先，循环解析注解时添加进集合的回滚元素。并递归调用RollbackRuleAttribute 的 getDepth 方法，如果这个异常的名字和注解中的异常名字匹配，则返回该异常的回滚类型。最后判断，如果没有匹配到，则调用父类的 rollbackOn 方法，如果匹配到了，并且该属性类型不是 NoRollbackRuleAttribute 类型，返回true。表示匹配到了，可以回滚。那么父类的 rollbackOn 方法肯定就是默认的回滚方法了。

这是父类的 rollbackOn 方法：

![](http://upload-images.jianshu.io/upload_images/4236553-c8ee8a9ccbb2db92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法判断，该异常如果是 RuntimeException 类型异常或者 是 Error 类型的，就回滚。这就是默认的回滚策略。

那么我们的方法肯定是匹配的 RuntimeException 异常，就会执行下面的方法。

![](http://upload-images.jianshu.io/upload_images/4236553-1132805e31316f64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，这行代码就是执行了我们的事务管理器的 rollback 方法，并且携带了事务状态对象。该方法实现在抽象类 AbstractPlatformTransactionManager 中，调用了自身的 processRollback 方法做真正的实现。

![](http://upload-images.jianshu.io/upload_images/4236553-5a6f1d12d7985007.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法首先切换事务状态，其实就是关闭SqlSession。

![](http://upload-images.jianshu.io/upload_images/4236553-3eccb847363718d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后调用 doRollback 方法。

![](http://upload-images.jianshu.io/upload_images/4236553-1a468d7075f82f75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先，从状态对象中获取数据库连接持有对象，然后获取数据库连接，调用 Connection 的 rollback 方法，也就是我们学习JDBC 时使用的方法。最后修改事务的状态。

到这里，事务的回滚就结束了。

那么，事务时如何提交的呢？

## 6. TransactionInterceptor 的 commitTransactionAfterReturning 方法（事务如何提交）。

![](http://upload-images.jianshu.io/upload_images/4236553-04056214359dca61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法简单的调用了事务管理器的 commit 方法。

AbstractPlatformTransactionManager 的 commit 方法。

![](http://upload-images.jianshu.io/upload_images/4236553-ac608644a6239ef7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先判断了事务的状态，如果状态不匹配，则调用回滚方法。如果状态正常，执行 processCommit 方法。该方法很长，楼主只截取其中一段：

![](http://upload-images.jianshu.io/upload_images/4236553-8acc2e2b6922d68a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先，commit 之前做一些状态切换工作。最重要的是执行 doCommit 方法，如果异常了，则回滚。那么 DataSourceTransactionManager 的 doCommit 是如何执行的呢？

![](http://upload-images.jianshu.io/upload_images/4236553-63a241cf81f99525.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，底层也是调用 JDBC 的 Connection 的 commit 方法。

到这里，我们就完成了数据库的提交。

## 7. 事务运行之前做了哪些工作？

从前面的分析，我们已经知道了事务是如何运行的，如何回滚的，又是如何提交的。在这是交互型的框架里，事务系统肯定做了很多的准备工作，同时，我们留下了很多的疑问，比如事务管理器从何而来？ TransactionAttributeSource  属性何时生成？AnnotationTransactionAttributeSource  构造什么时候调用？

我们一个个的来解释。

在Spring 中，有一个现成的类，ProxyTransactionManagementConfiguration，我们看看该类：

![](http://upload-images.jianshu.io/upload_images/4236553-1282b126083060ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看到这个类，应该可以解开我们的疑惑，这个类标注了配置注解，会在IOC的时候实例化该类，而该类中产生了几个Bean，比如事务拦截器 TransactionInterceptor，创建了 AnnotationTransactionAttributeSource 对象，并向事务拦截器添加了事务管理器。最后，将事务拦截器封装成通知器。那么，剩下最后一个问题就是，事务管理器从何而来？答案是他的父类 AbstractTransactionManagementConfiguration ：

![](http://upload-images.jianshu.io/upload_images/4236553-fcb24dd388fceea5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该类也是个配置类，自动注入了 TransactionManagementConfigurer 的配置集合，而并且寻找了配置 EnableTransactionManagement 注解的类，而我们在我们的项目中就是按照这个标准来实现的：

![](http://upload-images.jianshu.io/upload_images/4236553-28d41758c9d322bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们关联这两个类就能一目了然，Spring在启动的时候，会加载这两个配置类，在对 AbstractTransactionManagementConfiguration 的 setConfigurers 方法进行注入的时候，会从容器中找到对应类型的配置，并调用配置类的 annotationDrivenTransactionManager 方法，也就是我们实现的方法，获取到我们创建的 DataSourceTransactionManager 类。这样，我们的事务拦截器相关的类就完成了在Spring中的依赖关系。

但是，这个时候Spring中的事务运行还没有搭建完成。比如什么时候创建类的代理？根据什么创建代理，因为我们知道，Spring 中的事务就是使用AOP来完成的，必须使用动态代理或者 Cglib 代理来对目标方法进行拦截。

这就要复习我们之前的Spring IOC 的启动过程了。Spring 在创建bean的时候，会对每个Bean 的所有方法进行遍历，如果该方法匹配系统中任何一个拦截器的切点，就创建一个该Bean的代理对象。并且会将对应的通知器放入到代理类中。以便在执行代理方法的时候进行拦截。

具体代码步骤楼主贴一下：

1. 在对bean 进行初始化的时候会执行 AutowireCapableBeanFactory 接口的 applyBeanPostProcessorsAfterInitialization 的方法，其中会遍历容器中所有的bean后置处理器，后置处理器会调用 postProcessAfterInitialization 方法对bean进行处理。

![](http://upload-images.jianshu.io/upload_images/4236553-6d44b133cc6edd7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 在处理过程中，对bean 进行包装，也就是代理的创建，调用 getAdvicesAndAdvisorsForBean 方法，该方法会根据bean的信息获取到对应的拦截器并创建代理，创建代理的过程我们之前已经分析过了，不再赘述。

![](http://upload-images.jianshu.io/upload_images/4236553-0a490ac3757e1051.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 寻找匹配拦截器过程：首先找到所有的拦截器，然后，根据bean的信息进行匹配。

![](http://upload-images.jianshu.io/upload_images/4236553-923c9504b34b97bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4. 匹配的过程就是，找到目标类的所有方法，遍历，并调用拦截器的方法匹配器对每个方法进行匹配。方法匹配器就是事务拦截器中的 BeanFactoryTransactionAttributeSourceAdvisor 类，该类封装了 AnnotationTransactionAttributeSource 用于匹配事务注解的匹配器。

![](http://upload-images.jianshu.io/upload_images/4236553-66a202405bbbd189.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/4236553-2465b95ec883d884.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5. 最终调用方法匹配器中封装的注解解析器解析方法，判断方法是否含有事务注解从而决定是否生成代理：

![](http://upload-images.jianshu.io/upload_images/4236553-935654e3a9876b64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

到这里，就完成了所有事务代理对象的创建。

项目中的每个Bean都有了代理对象，在执行目标方法的时候，代理类会查看目标方法是否匹配代理中拦截器的方法匹配器中定义的切点。如果匹配，则执行拦截器的拦截方法，否则，直接执行目标方法。这就是含有事务注解和不含有事务注解方法的执行区别。


到这里，我们还剩下最后一个问题，我们知道，在分析mybatis 的时候，mybatis 也有自己的事务管理器，那么他们融合之后，他们的事务管理权在谁的手上，又是根据什么切换的呢？



## 8. mybatis 和 Spring 的事务管理权力之争

我们之前说过，在Spring中，mybatis 有 SqlSessionTemplate 代理执行，其实现类动态代理的 InvocationHandler 方法，那么最重要的方法就是 invoke 方法，其实这个方法我们已经看过了，今天再看一遍：

![](http://upload-images.jianshu.io/upload_images/4236553-0d8c648dd2f915a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们今天重点关注是否提交（报错肯定回滚），其中红框标出来的 if 判断，就是判断这个事务到底是Spring 来提交，还是 mybatis 来提交，那么我们看看这个方法 isSqlSessionTransactional ：

![](http://upload-images.jianshu.io/upload_images/4236553-be3d8eadd431999d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

该方法从Spring 的容器中取出持有 SqlSession 的 持有类，判断Spirng 持有的 SqlSession 和 Mybatis 持有的是否是同一个，如果是，则交给Spring，否则，Mybatis 自己处理。可以说很合理。

## 总结

今天的这篇文章可以说非常的长，我们分析了 SpringBoot 的事务运行过程，事务环境的搭建过程，mybatis 的事务和 Spring 事务如何协作。知道了整个事务其实是建立在AOP的基础之上，其核心类就是 TransactionInterceptor，该类就是 invokeWithinTransaction 方法是就事务处理的核心方法，其中封装了我们创建的 DataSourceTransactionManager 对象，该对象就是执行回滚或者提交的执行单位 其实，TransactionInterceptor 和我们平时标注 @Aspect 注解的类的作用相同，就是拦截指定的方法，而在 
TransactionInterceptor  中是通过是否标有事务注解来决定的。如果一个类中任意方法含有事务注解，那么这个方法就会被代理。而Mybatis 的事务和Spring 的事务协作则根据他们的SqlSession 是否是同一个SqlSession 来决定的，如果是同一个，则交给Spring，如果不是，Mybatis 则自己处理。

通过阅读源码，我们已经弄清楚了SpirngBoot 整个事务的运行过程。实际上，Spring 的其他版本也大同小异。底层都是 TransactionInterceptor ，只不过入口不一样。我相信，在以后的工作中，如果遇到了Spring事务相关的问题，再也不会感到无助了，因为知道了原理，可以深入到源码中查看。

到这里，楼主的 Spring ，mybatis ，Tomcat 的源码阅读之路暂时就告一段落了。源码只要领会精华即可。还有其他的知识需要花费更多的时间学习。比如并发，JVM.

good luck！！！！










