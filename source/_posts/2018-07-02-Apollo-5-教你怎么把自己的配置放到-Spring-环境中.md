---
layout: post
title: Apollo-5-教你怎么把自己的配置放到-Spring-环境中
date: 2018-07-02 11:11:11.000000000 +09:00
---
目录：
0. 前言
1. 处理方案
2. 简单例子

## 前言

有的时候，你可能需要在 Spring 环境中放入一些配置，但这些配置无法写死在配置文件中，只能运行时放入。那么，这个时候该怎么办呢？

Apollo  就是搞配置的，那么自然会遇到这个问题，他是如何处理的呢？

## 处理方案

首先要知道 Spring 环境中，一个配置的数据结构是什么？

是抽象类 `PropertySource<T> `， 内部是个 key value 结构。这个 T 可以是任意类型，取决于子类的设计。

子类可以通过重写 getProperty 抽象方法获取配置。

Spring 自身的 org.springframework.core.env.MapPropertySource 就重写了这个方法。

```java
public class MapPropertySource extends EnumerablePropertySource<Map<String, Object>> {

	public MapPropertySource(String name, Map<String, Object> source) {
		super(name, source);
	}


	@Override
	public Object getProperty(String name) {
		return this.source.get(name);
	}

	@Override
	public boolean containsProperty(String name) {
		return this.source.containsKey(name);
	}

	@Override
	public String[] getPropertyNames() {
		return StringUtils.toStringArray(this.source.keySet());
	}
}
``` 

可以看到，他的泛型是 Map，getProperty 方法则是从 Map 中获取。

**Apollo 就直接利用了这个类。**

![](https://upload-images.jianshu.io/upload_images/4236553-e67471ed5a73e2b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

两个不同的子类，不同的刷新逻辑。我们暂时不关心他们的不同。

这两个类都会被 RefreshableConfig 组合，添加到 Spring 的环境中。

```java
import org.springframework.core.env.ConfigurableEnvironment;

public abstract class RefreshableConfig {

  @Autowired
  private ConfigurableEnvironment environment; // Spring 环境

  @PostConstruct
  public void setup() {
  // 省略代码
    for (RefreshablePropertySource propertySource : propertySources) {
      propertySource.refresh();
      // 注意：成功刷新后，放到 Spring 的环境中
      environment.getPropertySources().addLast(propertySource);
    }
  // 省略代码
```

当从 Spring 的环境中获取配置的时候，具体代码是下面这样的：

```java
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
		for (PropertySource<?> propertySource : this.propertySources) {
            // 注意：这里调用的就是 propertySource.getProperty 方法，子类刚刚重写的方法
			Object value = propertySource.getProperty(key);
             // 省略无关代码........
			return convertValueIfNecessary(value, targetValueType);
		}
	    return null;
}
````

Spring 维护了一个 PropertySource 的集合，这个结合是有顺序的，也就是说，排在最前面的优先级最高（遍历从下标 0 开始）。

而用户可以在 PropertySource 里，维护一个配置字典（Map），这样，就类似 2 维数组的这样一个数据结构。

所以，配置是可以重名的，重名时，以最前面的 PropertySource 中的配置为准。所以，Spring 留给了几个 API：
1. addFirst(PropertySource<?> propertySource)
2. addLast(PropertySource<?> propertySource)
3.  addBefore(String relativePropertySourceName, PropertySource<?> propertySource)
4. addAfter(String relativePropertySourceName, PropertySource<?> propertySource) 

从名字可以看出，通过这些 API，我们可以将 propertySource 插入到我们指定的地方。从而可以手动控制配置的优先级。

Spring 中有个现成的 CompositePropertySource 类，内部聚合了一个 PropertySource Set 集合，当 getProperty(String name) 的时候，就会遍历这个集合，然后调用这个 propertySource 的 getProperty(name)  方法。相当于 3 维数组。


大概的设计是这样：

![](https://upload-images.jianshu.io/upload_images/4236553-bf02da7a86e22149.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一个环境中，有多个 PS（PropertySource 简称），每个 PS 可以直接包含配置，也可以再包装一层 PS。

## 简单例子

我们这里有个简单的例子，需求：
程序里有个配置，但不能写死在配置文件中，只能在程序启动过程中进行配置，然后注入到 Spring 环境中，让 Spring 在之后的 IOC 中，可以正常的使用这些配置。

代码如下：

```java
@SpringBootApplication
public class DemoApplication {

  @Value("${timeout:1}")
  String timeout;


  public static void main(String[] args) throws InterruptedException {
    ApplicationContext c = SpringApplication.run(DemoApplication.class, args);
    for (; ; ) {
      Thread.sleep(1000);
      System.out.println(c.getBean(DemoApplication.class).timeout);
    }
  }
}
```

application.properties 配置文件

```
timeout=100
```

上面的代码中，我们在 bean 中定义了一个属性 timeout， 并在本地配置文件中写入了一个 100 的值，也在表达式中给了一个默认值 1。

那么现在打印出来的就是配置文件中的值：100.

但是，这不是我们想要的结果，所以需要修改代码。

我们加入一个类：

```java
@Component
class Test implements EnvironmentAware, BeanFactoryPostProcessor {

  @Override
  public void setEnvironment(Environment environment) {
    ((ConfigurableEnvironment) environment).getPropertySources()
        // 这里是 addFirst,优先级高于 application.properties 配置
        .addFirst(new PropertySource<String>("timeoutConfig", "12345") {
          // 重点
          @Override
          public Object getProperty(String s) {
            if (s.equals("timeout")) {//
              return source;// 返回构造方法中的 source :12345
            }
            return null;
          }
        });
  }

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
      throws BeansException {
    // NOP
  }
}
```

运行之后，结果：12345

```java
2018-07-02 15:26:54.315  INFO 43393 --- [           main] o.s.j.e.a.AnnotationMBeanExporter        : Registering beans for JMX exposure on startup
2018-07-02 15:26:54.327  INFO 43393 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 0.991 seconds (JVM running for 1.49)
12345
12345
```

为什么加入了这个类，就能够代替配置文件中的属性呢？解释一下这个类的作用。

我们要做的事情就是在 Spring 的环境中，插入自定义的 PS 对象，以便容器获取的时候，能够通过 getProperty 方法获取对应的配置。

所以，我们要拿到 Spring 环境对象，还需要创建一个 PS 对象，并重写 getProperty 方法，同时，注意：自己的 PS 配置优先级需要高于容器配置文件的优先级，保险起见，放在第一位。

PS 构造方法的第一个参数没什么用，就是一个标识符，第二个参数就是 source，可以定义为任何类型，String，Map，都可以，我们这里简单期间，就是一个 String，直接返回这个值，如果是 Map，就调用 Map 的 get 方法。

为什么要实现 BeanFactoryPostProcessor 接口呢？ 实现 BeanFactoryPostProcessor 接口的目的是让该 Bean 的加载时机提前，高于目标 Bean 的初始化。否则，目标 Bean 中的 timeout 属性都注入结束了，后面的操作就没有意义了。


# 总结

说白了，就是不想写配置文件！！！ 

而且也不想改老项目的代码，老项目即使在删除配置文件的情况下，依然能够使用配置中心！

这就需要熟悉 Spring 的配置加载逻辑和属性获取逻辑。

现在，我们知道，只需要拿到 Spirng 的环境对象，并向环境中添加自定义的  PS 对象，重写 PS 的 getProperty 方法，即可获取配置（注意优先级）。

还需要注意加载这个配置的 bean 的优先级也要很高，通常实现 BeanFactoryPostProcessor 接口就足够了，如果还不够，就需要做一些特殊操作。


















































  
