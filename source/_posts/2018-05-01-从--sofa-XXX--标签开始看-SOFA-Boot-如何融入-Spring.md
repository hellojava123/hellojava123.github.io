---
layout: post
title: 从--sofa-XXX--标签开始看-SOFA-Boot-如何融入-Spring
date: 2018-05-01 11:11:11.000000000 +09:00
---
![](https://upload-images.jianshu.io/upload_images/4236553-006332d17fbdcca5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 前言

SOFA-Boot  现阶段支持 XML 的方式在 Spring 中定义 Bean，通过这些标签，我们就能从 Spring 容器中取出 RPC 中的引用，并进行调用，那么他是如何处理这些自定义标签的呢？一起来看看。

## 如何使用？

官方例子：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:sofa="http://sofastack.io/schema/sofaboot"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://sofastack.io/schema/sofaboot   http://sofastack.io/schema/sofaboot.xsd"
       default-autowire="byName">

    <bean id="personServiceImpl" class="com.alipay.sofa.boot.examples.demo.rpc.bean.PersonServiceImpl"/>

    <sofa:service ref="personServiceImpl" interface="com.alipay.sofa.boot.examples.demo.rpc.bean.PersonService">
        <sofa:binding.bolt/>
        <sofa:binding.rest/>
    </sofa:service>


    <sofa:reference id="personReferenceBolt" interface="com.alipay.sofa.boot.examples.demo.rpc.bean.PersonService">
        <sofa:binding.bolt/>
    </sofa:reference>

    <sofa:reference id="personReferenceRest" interface="com.alipay.sofa.boot.examples.demo.rpc.bean.PersonService">
        <sofa:binding.rest/>
    </sofa:reference>

    <bean id="personFilter" class="com.alipay.sofa.boot.examples.demo.rpc.bean.PersonServiceFilter"/>

</beans>
```

显眼的 sofa 标签。那么如何知道他是怎么处理这些标签的内容的呢？

答：从上面的 xmlns 命名空间可以找到。如果写过自定义标签的话，一定很熟悉了。

如果没有写过的话，就简单介绍一下。Spring 支持用户自定标签，需要遵守以下规范。
1. 编写 xsd 文件，就是定义标签的属性。

2. 继承抽象类 `NamespaceHandlerSupport` 并实现 init 方法，此类就是处理命名空间的类，不仅需要实现 init 方法，你还需要继承 `AbstractSingleBeanDefinitionParser` 类，自行解析标签。通常写法是这样：
`registerBeanDefinitionParser("tagName",new UserBeanDefinitionParser());`

3. 在做完上面的步骤后，你需要在 META-INF 目录下编写 `spring.handlers` 和 `spring.schemas` 文件，前者 key 为命名空间名称，value 为 `NamespaceHandlerSupport` 的具体实现；后者 key 为 xsd 命名空间，value 为 xsd 具体文件（classpath 下）。

完成上面的步骤后，Spring 在加载 xml 配置文件的时候，会检查命名空间，如果是自定义的，则会根据命名空间的 key 找到对应的解析器，也就是 `NamespaceHandlerSupport` 对自定义标签进行解析。

So，我们的目的是看 sofa 标签是如何解析的，则找到解析 sofa 的`NamespaceHandlerSupport`。

## 寻找解析 sofa 标签的源头

通过 IDEA 或者别的工具全局搜索，我们找到了源码中位于 start 模块下，resources 下的 META-INF 目录，目录下有以下几个文件：

1. rpc.xsd
2. sofaboot.xsd
3. spring.handlers
4. spring.schemas

这几个文件我们是比较关注的，重点看 handlers 文件。内容如下：

```xml
http\://sofastack.io/schema/sofaboot=com.alipay.sofa.infra.config.spring.namespace.handler.SofaBootNamespaceHandler
```

`SofaBootNamespaceHandler` 明显就是明明空间处理类，继承了 Spring 的 `NamespaceHandlerSupport`。

该类的 init 方法通过 Java 的 SPI 进行扩展，找到 SofaBootTagNameSupport 的标签支持类，目前有 2 个实现： ServiceDefinitionParser 和 ReferenceDefinitionParser。

两个类支持不同的 element。一个是引用服务，一个是发布服务。

这样就比较清晰了。在得到两个类之后，注册到 Spring 的 parsers map 中。key 是 element 名字，value 是解析器。

具体的解析上面说了，service 和 reference。

Sofa 在这个设计上使用了模板模式，使用一个抽象类 `AbstractContractDefinitionParser`，并定义一个 doParseInternal 抽象方法让子类去实现。

抽象父类将一些公用的属性进行解析，下图中都是公用的属性：

![image.png](https://upload-images.jianshu.io/upload_images/4236553-03c5b80b0b9be698.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你咋确定是公用的属性呢？首先，xml 解析后，肯定是要注入到 Bean 里面去的，那么这个 Bean 是什么呢？实际上，在 `AbstractSingleBeanDefinitionParser`  抽象类中，会有一个返回 Bean 类型的方法，对应着一个 tag，而他们的父类就是`  AbstractContractFactoryBean`。

该类公用属性就对应上面图片中的定义。

有了这些基础的信息，还是不能够直接使用的，毕竟都是字符串。我们要看看 SOFA 是如何将自己和 Spring 容器融合在一起的。

## 如何融入 Spring？

来看看他们的子类， ReferenceDefinitionParser 类（ServiceFactoryBean 类似）。对应的  Bean 是 ReferenceFactoryBean，该类单独定义了负载均衡（loadBalance）的属性。

啊，从这个类的名字看，很熟悉，之前在分析 Spring 源码的时候，就看见过 FactoryBean。可以通过 getObject 方法修改返回的 Bean。

来看看这个类的 类图

![image.png](https://upload-images.jianshu.io/upload_images/4236553-bcc952b2247a1059.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如我们所料，继承了 Spring 中关键的扩展点。而他的 getObject 方法将返回一个代理：

```java
   @Override
    public Object getObject() throws Exception {
        return proxy;
    }
```

具体创建代理的方法是 `com.alipay.sofa.runtime.service.component.ReferenceComponent` 类的 createProxy 方法，然后调用适配器。适配器的作用就是胶水的作用，将 Spring 容器和第三方框架粘合起来，Spring 提供的接口则是 FactoryBean 等接口，而第三方框架可以在 getObject 方法中大有作为。

SOFA-Boot 的适配器在 `om.alipay.sofa.rpc.boot.runtime.adapter` 包中，具体实现则是 RpcBindingAdapter  类，其中 outBinding 方法用于发布服务，inBinding 方法用于引用服务，这里直接使用的就是 SOFA-RPC 的 consumerConfig 和 providerConfig。这就应该很熟悉了吧。哈哈。

来点代码看看(已去除异常处理)：

```java
@Override
public Object outBinding(Object contract, RpcBinding binding, Object target, SofaRuntimeContext sofaRuntimeContext) {

    String uniqueName = ProviderConfigContainer.createUniqueName((Contract) contract, binding);
    ProviderConfig providerConfig = ProviderConfigContainer.getProviderConfig(uniqueName);

    providerConfig.export();

    if (ProviderConfigContainer.isAllowPublish()) {
        Registry registry = RegistryConfigContainer.getRegistry();
        providerConfig.setRegister(true);
        registry.register(providerConfig);
    }
    return Boolean.TRUE;
}
```

```java
@Override
public Object inBinding(Object contract, RpcBinding binding, SofaRuntimeContext sofaRuntimeContext) {
    ConsumerConfig consumerConfig = ConsumerConfigHelper.getConsumerConfig((Contract) contract, binding);
    ConsumerConfigContainer.addConsumerConfig(binding, consumerConfig);

    Object result = consumerConfig.refer();
    binding.setConsumerConfig(consumerConfig);
    return result;
}
```

So，当 Spring 容器使用 getObject 方法的时候，获取的就是这个代理对象啦。是不是很简单?

有一点需要注意一下，ServiceFactoryBean 的设计和 ReferenceFactoryBean 确实是类似，但是！但是！他的 getObject 方法就是实现类本身，这是 RPC 框架本身的设计决定的，因为它不需要代理，只需要发布服务就行，当收到了客户端传来的信息，就直接调用实现类的指定方法就好了，没有客户端这么复杂。

当然，SOFA 中还有一个组件的概念，我们有时间会好好看看这块的设计。


## 总结

这次我们从 SOFA-Boot 配置文件的 xml 标签开始，对他如何融入 Spring 进行了分析，实际上，整体和我们联想的类似，使用 Spring 的 FactoryBean 的 getObject 方法返回代理，如果是客户端的话，在 getObject 方法中，会创建一个动态代理，这就要使用 SOFA-RPC 了，所以，融合 RPC 和 Spring 的任务肯定是个适配器。SOFA 的实现就是 RpcBindingAdapter 类。在该类中，将 RPC 和 Spring 适配。

而发布服务相比较引用服务就简单一点了，整体上就是通过注解将服务发布，当然也是使用的 RpcBindingAdapter 进行适配。但是没有使用代理（不需要）。

好了，这次研究 SOFA 和 Spring 融合的过程就到这里啦。

bye ！








