---
layout: post
title: 深入理解Spring-之-源码剖析AOP（XML配置方式）
date: 2017-12-07 11:11:11.000000000 +09:00
---
Spring 的两大核心，一是IOC，我们之前已经学习过，并且已经自己动手实现了一个，而令一个则是大名鼎鼎的 AOP，AOP的具体概念我就不介绍了，我们今天重点是要从源码层面去看看 spring 的 AOP 是如何实现的。注意，今天楼主给大家分享的是 XML 配置AOP的方式，不是我们经常使用的注解方式，为什么呢？

有几个原因：
1. Spring AOP 在 2.0 版本之前都是使用的 XML 配置方式，封装的层次相比注解要少，对于我们学习AOP是个很好的例子。
2. 虽然现在是2017年，现在使用SpringBoot 都是使用注解了， 但是底层原理都是一样的，只不过多了几层封装。当然，我们也是要去扒开它的源码的。但不是今天。
3. 楼主也还没有分析注解方式的AOP。-_-|||

我们主要分为几个步骤去理解：
1. 查看源码了解 spring AOP 的接口设计。Advice，PointCut，Advisor。
2. 用一个最简单的代码例子 debug 追踪源码。


那么我们现看第一步：
## 1. Spring AOP 接口设计

### 1.1 PointCut （连接点，定义匹配哪些方法）
我们打开 Spring 的源码，查看 PointCut 接口设计：

```java
public interface Pointcut {
	ClassFilter getClassFilter();
	MethodMatcher getMethodMatcher();
	Pointcut TRUE = TruePointcut.INSTANCE;
}

```

该接口定义了2 个方法，一个成员变量。我们先看第一个方法， `ClassFilter getClassFilter()` ，该方法返回一个类过滤器，由于一个类可能会被多个代理类代理，于是Spring引入了责任链模式，另一个方法则是 `MethodMatcher getMethodMatcher()` ，表示返回一个方法匹配器，我们知道，AOP 的作用是代理方法，那么，Spirng 怎么知道代理哪些方法呢？必须通过某种方式来匹配方法的名称来决定是否对该方法进行增强，这就是 MethodMatcher  的作用。还有要给默认的 Pointcut 实例，该实例对于任何方法的匹配结果都是返回 true。

我们关注一下 MethodMatcher 接口：

```java
public interface MethodMatcher {
	boolean matches(Method method, @Nullable Class<?> targetClass);
	boolean isRuntime();
	boolean matches(Method method, @Nullable Class<?> targetClass, Object... args);
	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}
```
该接口定义了静态方法匹配器和动态方法匹配器。所谓静态方法匹配器，它仅对方法名签名（包括方法名和入参类型及顺序）进行匹配；而动态方法匹配器，会在运行期检查方法入参的值。静态匹配仅会判别一次，而动态匹配因为每次调用方法的入参都可能不一样，所以每次都必须判断。一般情况下，动态匹配不常用。方法匹配器的类型由isRuntime()返回值决定，返回false表示是静态方法匹配器，返回true表示是动态方法匹配器。

总的来说， PointCut 和 MethodMatcher 是依赖关系，定义了AOP应该匹配什么方法以及如何匹配。

### 1.2 Advice (通知，定义在链接点做什么)

注意，Advice 接口只是一个标识，什么也没有定义，但是我们常用的几个接口，比如 BeforeAdvice，AfterAdvice，都是继承自它。我们关注一下 AfterAdvice 的子接口 AfterReturningAdvice ：

```java
public interface AfterReturningAdvice extends AfterAdvice {
	void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;
}

````

该接口定义了一个方法，afterReturning，参数分别是返回值，目标方法，参数，目标方法类，在目标方法执行之后会回调该方法。那么我们就可以在该方法中执行我们的切面逻辑，BeforeAdvice 也是一样的道理。

### 1.3 Advisor (通知器，将 Advice 和 PointCut 结合起来)

有了对目标方法的增强接口 Advice 和 如何匹配目标方法接口 PointCut 接口后，那么我们就需要用一个对象将他们结合起来，发挥AOP 的作用，所以Spring 设计了 Advisor（通知器），经过我们刚刚的描述，我们应该知道了，这个 Advisor 肯定依赖了 Advice 和 PointCut，我们看看接口设计：

```java
public interface Advisor {

	Advice EMPTY_ADVICE = new Advice() {};

	Advice getAdvice();

	boolean isPerInstance();

}

```

还有他的子接口：

```java
public interface PointcutAdvisor extends Advisor {

	Pointcut getPointcut();
}

```
最重要的两个方法，getAdvice，getPointcut。和我们预想的基本一致。


接下来，我们可以停下来思考一下，现在有了这个东西，我们怎么实现面向切面编程；
1. 首先我们需要告诉AOP在哪里进行切面。比如在某个类的方法前后进行切面。
2. 告诉AOP 切面之后做什么，也就是说，我们知道了在哪里进行切面，那么我们也该让spring知道在切点处做什么。
3. 我们知道，Spring AOP 的底层实现是动态代理（不管是JDK还是Cglib），那么就需要一个代理对象，那么如何生成呢？

接下来，我们将通过代码的方式，解答这三个疑惑。

## 2. 从一个简单的AOP例子

首先，我们需要实现刚刚我们说的3个接口，还有一个目标类，还要一个配置文件。一个一个来。

### 2.1. Pointcut  接口实现

```java
package test;

import java.lang.reflect.Method;
import org.springframework.aop.ClassFilter;
import org.springframework.aop.MethodMatcher;
import org.springframework.aop.Pointcut;

public class TestPointcut implements Pointcut {

	@Override
	public ClassFilter getClassFilter() {
		return ClassFilter.TRUE;
	}

	@Override
	public MethodMatcher getMethodMatcher() {
		return new MethodMatcher() {

			public boolean matches(Method method, Class<?> targetClass, Object[] args) {
				if (method.getName().equals("test")) {
					return true;
				}
				return false;
			}

			public boolean matches(Method method, Class<?> targetClass) {
				if (method.getName().equals("test")) {
					return true;
				}
				return false;
			}

			public boolean isRuntime() {
				return true;
			}
		};
	}

}

```
我们如何定义匹配？只要方法名称是test则对该方法进行增强或者说拦截。

### 2.2 AfterAdvice 实现

```java
package test;

import java.lang.reflect.Method;
import org.springframework.aop.AfterReturningAdvice;

public class TestAfterAdvice implements AfterReturningAdvice {

	@Override
	public void afterReturning(Object returnValue, Method method,
			Object[] args, Object target) throws Throwable {
		System.out.println(
				"after " + target.getClass().getSimpleName() + "." + method.getName() + "()");
	}

}

```

我们在方法执行完毕后打印该方法的名称和该目标类的名称。这就是我们做的简单增强。

### 2.3 Advisor 通知器的实现

```java
package test;

import org.aopalliance.aop.Advice;
import org.springframework.aop.Pointcut;
import org.springframework.aop.PointcutAdvisor;

/**
 * 通知器
 */
public class TestAdvisor implements PointcutAdvisor {

	/**
	 * 获取通知处理逻辑
	 */
	@Override
	public Advice getAdvice() {
		return new TestAfterAdvice();
	}

	@Override
	public boolean isPerInstance() {
		return false;
	}

	/**
	 * 获取切入点
	 */
	@Override
	public Pointcut getPointcut() {
		return new TestPointcut();
	}

}

```

我们实现了 PointcutAdvisor 接口，返回我们刚才定义的两个类。完成了他们的组合。

### 2.4 定义目标类 Targe

```java
package test;

public class TestTarget {

	public void test() {
		System.out.println("target.test()");
	}

	public void test2() {
		System.out.println("target.test2()");
	}
}

```

该目标的实现和简单，就是2个方法，分别打印自己的名字。

### 2.5 定义XML 配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd">

	<bean id="testAdvisor" class="test.TestAdvisor"></bean>
	<bean id="testTarget" class="test.TestTarget"></bean>
	<bean id="testAOP" class="org.springframework.aop.framework.ProxyFactoryBean">
		<property name="targetName">
			<value>testTarget</value>
		</property>
		<property name="interceptorNames">
			<list>
				<value>testAdvisor</value>
			</list>
		</property>
	</bean>

</beans>


```

可以看到，我们定义了3个bean，上面两个是我们刚刚定义的，下面一个我们要好好说说，ProxyFactoryBean 是一个什么东西呢？首先他是一个 FactoryBean，我们在学习 IOC 的时候知道， FactoryBean 是Spring 留给我们扩展用的，实现该接口的类可以自定类的各种功能。ProxyFactoryBean 当然也实现了自己的很多自定义功能。ProxyFactoryBean 也是Spring IOC 环境中创建AOP 应用的底层方法，Spring 正式通过它来实现对AOP的封装。这样我们更加接近Spring的底层设计。而该类需要注入两个属性一个目标类，一个拦截类，ProxyFactoryBean 会生成一个动态代理类来完成对目标方法的拦截。

### 2.6 测试类

```java
package test;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.FileSystemXmlApplicationContext;

public class TestAOP {

	public static void main(String[] args) {
		ApplicationContext applicationContext = new FileSystemXmlApplicationContext(
				"spring-context/src/test/java/test/beans.xml");
		TestTarget target = (TestTarget) applicationContext.getBean("testAOP");
		target.test();
		System.out.println("----------------");
		target.test2();
	}

}

```

我们看看运行结果：

![](http://upload-images.jianshu.io/upload_images/4236553-9df3c2c91c3fff9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到结果符合我们的预期，因为我们只配置了在test名称的方法之后打印该方法的名称和该目标类的名称，而test2 则没有配置，因此也就没有打印。

那么是怎么实现的呢？ 让我们进入源码看个究竟。

## 3. 深入 AOP 源码实现

首先我们看看我们的测试代码，我们第一句代码是IOC 初始化，这个我们就不讲了，我们在之前的文章已经分析过，我们重点看第二行代码，我们debug 到第三行，看看第二行返回的对象是什么？

![](http://upload-images.jianshu.io/upload_images/4236553-4cfca549cde2acc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到，第二行从IOC容器中取出的是一个Cglib 生成的代理对象，也既是继承了我们TestTarge 类的实例，而不是我们在 XML 中定义的 ProxyFactoryBean 对象，也就是说， FactoryBean 确实能够在IOC容器中做一些定制化。那么我们就很好奇，这个代理对象是怎么生成的。我们从第二行代码开始进入。

首先进入抽象类 AbstractApplicationContext 的getBean 方法，从容器或获取 Bean，再调用 doGetBean 方法，这个方法我们很熟悉，因为再之前的IOC过程中，我们看了好多遍了，我们重点看看该方法实现：

```java
	@SuppressWarnings("unchecked")
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();// 设置依赖关系
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						getBean(dep);// 递归
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {// 创建bean
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);// 获取实例
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```
该方法会进入到第一个if块中的 `bean = getObjectForBeanInstance(sharedInstance, name, beanName, null)` 方法中，而 getObjectForBeanInstance 方法则会先判断缓存是否存在，如果不存在，则进入父类的 getObjectForBeanInstance 方法，我们看看该方法实现：

```java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd == null) {
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}

```
首先判断是否是 Bean 引用类型并且是否是 Factory 类型，很面向不是Bean 引用类型（Bean引用类型指的是IOC在解析XML文件 的时候，会有 ref 属性，而这个ref 对象还没有实例化，则暂时创建一个Bean引用类型的实例，用于在依赖注入的时候判断是否是Bean的属性类型，如果是，则从容器中取出，如果不是，则是基本类型，就直接赋值），然后进入下面的if判断，很明显会直接跳过。进入下面的 getCachedObjectForFactoryBean(beanName) 方法，从缓存中取出，很明显，第一次肯定返回null，继续向下，进入if块，重点在 object = getObjectFromFactoryBean(factory, beanName, !synthetic) 方法，我们进入该方法查看：

```java
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						if (shouldPostProcess) {
							try {
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
						}
						this.factoryBeanObjectCache.put(beanName, object);
					}
				}
				return object;
			}
		}
		else {
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}

```
该方法还是先重缓存中取出，然后进入  doGetObjectFromFactoryBean(factory, beanName) 方法，我们看看该方法：

```java
	private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
			throws BeanCreationException {

		Object object;
		try {
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () ->
							factory.getObject(), acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				object = factory.getObject();// 此处调用 ProxyFactoryBean
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully
		// initialized yet: Many FactoryBeans just return null then.
		if (object == null) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			object = new NullBean();
		}
		return object;
	}
```
该方法会直接进入  object = factory.getObject() 行，也就是 ProxyFactoryBean 的 getObject 方法，还记得我们说过，Spring 允许我们从写 getObject 方法来实现特定逻辑吗？ 我们看看该方法实现：

```java
	public Object getObject() throws BeansException {
		initializeAdvisorChain();// 为代理对象配置Advisor链
		if (isSingleton()) {// 单例
			return getSingletonInstance();
		}
		else {
			if (this.targetName == null) {
				logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
						"Enable prototype proxies by setting the 'targetName' property.");
			}// 非单例
			return newPrototypeInstance();
		}
	}
```

该方法很重要，首先初始化通知器链，然后获取单例，这里返回的就是我们最初看到的Cglib 代理。这里的初始化过滤器链的重要作用就是将连接起来，基本实现就是循环我们在配置文件中配置的通知器，按照链表的方式连接起来。具体代码各位有兴趣自己去看，今天这个不是重点。首先判断是否是单例，然后我们着重关注下面的方法 getSingletonInstance()；

```java
	private synchronized Object getSingletonInstance() {
		if (this.singletonInstance == null) {
			this.targetSource = freshTargetSource();
			if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
				// Rely on AOP infrastructure to tell us what interfaces to proxy.
				Class<?> targetClass = getTargetClass();
				if (targetClass == null) {
					throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
				}
				setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
			}
			// Initialize the shared singleton instance.
			super.setFrozen(this.freezeProxy);
			this.singletonInstance = getProxy(createAopProxy());
		}
		return this.singletonInstance;
	}
```
该方法是同步方法，防止并发错误，因为有共享变量。首先返回一个包装过的目标对象，然后是否含有接口，我们的目标类没有接口，进入if块，从目标类中取出所有接口并设置接口。很明显，并没有什么作用，然后向下走，重要的一行是 `getProxy(createAopProxy())`，先创建AOP，再获取代理。我们先看 crateAopProxy。

```java
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		return getAopProxyFactory().createAopProxy(this);
	}
```
该方法返回一个AopProxy 类型的实例，我们看看该接口：

```java
public interface AopProxy {
	Object getProxy();
	Object getProxy(@Nullable ClassLoader classLoader);
}

```

该接口定义了两个重载方法，我们看看它有哪些实现：

![](http://upload-images.jianshu.io/upload_images/4236553-ea75ed2f65c3a0d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这是该接口的继承图，分别是 JdkDynamicAopProxy 动态代理和 CglibAopProxy 代理。而 JdkDynamicAopProxy 实现了 InvocationHandler 接口，如果熟悉Java 动态代理，应该和熟悉该接口，实现了该接口的类并实现invoke方法，再代理类调用的时候，会回调该方法。实现动态代理。

我们继续看 createAopProxy 方法，该方法主要逻辑是创建一个AOP 工厂，默认工厂是 DefaultAopProxyFactory，该类的 createAopProxy 方法则根据 ProxyFactoryBean 的一些属性来决定创建哪种代理：

```java
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```
只要满足三个条件中的其中一个就，我们看最后一个判断，是否含有接口，没有则返回ture，也就是说，如果目标类含有接口，则创建Cglib 代理，否则则是JDK代理。最终创建了一个 ObjenesisCglibAopProxy 代理。

我们回到 ProxyFactoryBean 类的 getProxy 方法，当 createAopProxy 返回一个Cglib 代理的后，则调用 getProxy 方法获取一个代理对象，我们看看该方法实现：

```java
	@Override
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
		}

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
		catch (CodeGenerationException | IllegalArgumentException ex) {
			throw new AopConfigException("Could not generate CGLIB subclass of class [" +
					this.advised.getTargetClass() + "]: " +
					"Common causes of this problem include using a final class or a non-visible class",
					ex);
		}
		catch (Throwable ex) {
			// TargetSource.getTarget() failed
			throw new AopConfigException("Unexpected AOP exception", ex);
		}
	}

````
该方法基本上就是使用了Cglib 的库的一些API最后通过字节码生成代理类，比如 Enhancer 增强器，楼主对ASM 和 Cglib 也不是很熟悉，下次有机会再看详细实现。总之，我们已经知道了Spring 是如何生成代理对象的，主要的通过 ProxyFactoryBean 来实现。

最后，返回代理类，执行代理类的方法。完成切面编程。

## 4. 总结

我们通过一个简单的例子debug spring 源码，了解了通过配置文件方式配置AOP的详细过程，其中起最重要作用的还是 ProxyFactoryBean ，在定制化Bena的过程中起到了很大的作用，也提醒了我们，如果想在Spring的bean容器实现一些特别的功能，可以实现 FactoryBean 接口，自定义自己的需要Bean。还有一点，今天我们学习的例子是通过XML方式，而这个方式确实有些古老，虽然不妨碍我们学习 AOP 的精髓，但我们还是希望能够深入了解基于注解的AOP的具体实现，也许实现源码相似，但我们还是想知道到底有哪些不同，最起码楼主在刚刚的调试中发现最新的SpringBoot的AOP不是基于ProxyFactroyBean实现的。但不要灰心，原理都是相同的。剩下的就是我们自己去挖掘。



good luck！！！







