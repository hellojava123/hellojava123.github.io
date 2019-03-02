---
layout: post
title: Apollo-1--融合-Spring-的三个入口
date: 2018-06-30 11:11:11.000000000 +09:00
---
## 前言

Spring 作为 Java 世界非官方标准框架，任何一个中间件想要得到良好的发展，必须完美支持 Spring 的各种特性，即：无缝融入 Spring。

Apollo 作为分布式配置中心，服务于 Java 应用程序，Java 应用程序可以通过 Apollo 提供的 Client 获取远程配置信息。而如何将这个 Client 高度融合到用户的应用程序中呢？

这就需要针对 Spring 提供给我们的接口进行扩展。

在之前的文章中，已经大致聊过 Spring 的一些扩展接口：[深入理解Spring 之 Spring 进阶开发必知必会 之 Spring 扩展接口](https://www.jianshu.com/p/685ced8abe92)。

而想融入 Spring，首先得找到入口，然后才能注册相关的类进行自己系统的初始化。
所以，如何找到并处理入口成了一门学问，我们今天看看 apollo 如何处理的。

## 第一种入口：XML 

XML 是传统 Java 项目的配置文件，特别是 Spring MVC 项目。虽然现在都是使用的自动化配置，但仍然有一些遗留项目使用 XML，因此，支持 XML 是大部分中间件的必须工作。

支持 XML 需要准守 Spring 的几个约定：
1. 继承 `NamespaceHandlerSupport` 抽象类，重写 init  方法，调用 registerBeanDefinitionParser 方法，并传入自己继承的 `AbstractSingleBeanDefinitionParser` 子类用于解析标签，重写他的 getBeanClass 方法，返回一个 Bean，用于注册相关的 Bean。
2. classpath 下创建 META-INF 目录，创建 `spring.handlers` 文件，将 xml 配置中的 URL 指向 `NamespaceHandlerSupport`。
3. META-INF 目录下，创建 `apollo-1.0.0.xsd` xsd 文件，用于解释自定义标签。
4. META-INF 目录下，继续创建 `spring.schemas` 文件，将 xml 配置中的 xsd URL 指向  xsd 文件。

如果你的 xml 配置中，引用了 apollo 的标签，Spring 将会根据 xml 中的 URL 找到 `spring.handlers` 中的 `NamespaceHandlerSupport` 类，并对标签进行解析。也会从 getBeanClass 得到一个设置的 bean，在这个 bean 里，做了 apollo 关键类的注册。

## 第二种入口：@Import 注解

相对于基于XML的配置，基于Java的配置是目前比较流行的方式。

@Import 注解的使用方式：
1. 定义一个自己的启动注解。并标注 @import 注解, 其实就是 xml 中的  import 标签，在该注解中，可以配置一个类，这个类就会注册进 Spring 的容器，成为 Bean，你就可以在这个 bean 里做文章。
在 apollo 中，使用方式如下：
```java
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.TYPE) @Documented
@Import(ApolloConfigRegistrar.class)
public @interface EnableApolloConfig {
  String[] value() default {ConfigConsts.NAMESPACE_APPLICATION};
  int order() default Ordered.LOWEST_PRECEDENCE;
}
````

2. 从上面可以看出 `ApolloConfigRegistrar` 类是 apollo 注册进的 bean。这个 bean 用于处理 @EnableApolloConfig 注解，同时注册 apollo 关键 Bean 到 Spring  容器中。

3. 用户只需在 Spring 系统中的某个类上，标注 @EnableApolloConfig ，就可以通过 Spring 的方式（自动更新，注解等）使用 apollo 功能。

## 第三种入口：SpringBoot Starter

目前最流行的框架就是 Spring Boot ，兼容 SpringBoot 是一个大趋势。

Spring Boot 提供 `spring-boot-autoconfigure` 让第三方框架兼容 Boot，称之为 starter。

创建一个 starter 需要遵守几个约定：
1. maven 引入 `spring-boot-autoconfigure` artifact.
2. 创建一个类，实现`ApplicationContextInitializer` 接口，重写 initialize 方法，该方法在容器初始化的时候调用。
3. META-INF 创建 `spring.factories` 文件，Boot 启动时会自动扫描这个文件。需要在这个文件中写入一个步骤 2 创建的类，类似 `org.springframework.context.ApplicationContextInitializer=\com.ctrip.framework.apollo.spring.boot.ApolloApplicationContextInitializer`。这个类的作用是提前（容器初始化前）加载关键配置到 Spring 环境。
4. 在  `spring.factories`  文件中，还需要让 boot 的 `EnableAutoConfiguration` 自动配置类指向一个自定义类。**这是 SpringBoot  starter 的关键**，例如：`org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.ctrip.framework.apollo.spring.boot.ApolloAutoConfiguration`。ApolloAutoConfiguration 就会加入的 apollo 的配置 bean 中。你可以在这个配置 bean 中，创建一个关键 bean ，用于处理系统相关的初始化类。例如 apollo 的方式：
```java
@Configuration
@ConditionalOnProperty(PropertySourcesConstants.APOLLO_BOOTSTRAP_ENABLED)
@ConditionalOnMissingBean(PropertySourcesProcessor.class)// 当Spring Context中不存在该Bean时
public class ApolloAutoConfiguration {
  @Bean
  public ConfigPropertySourcesProcessor configPropertySourcesProcessor() {
    return new ConfigPropertySourcesProcessor();
  }
}
```
在 apollo 中，`ConfigPropertySourcesProcessor` 就是用来注册系统关键 bean 的。


## 总结

本文重点介绍了 3 种入口：
1. XML 方式，通过在 getBeanClass 方法返回系统关键 Bean。
2. @Import 注解，通过在注解中定义 Bean，然后在该 Bean 中处理。
3. SpringBoot Starter 方式，通过 `spring.factories` 文件中定义自动配置类，可以注册系统关键 bean。

在以后的开发中，如果想融入 Spring，就可以通过这 3 种方式自行处理。




























