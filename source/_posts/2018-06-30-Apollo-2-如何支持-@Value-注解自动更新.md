---
layout: post
title: Apollo-2-如何支持-@Value-注解自动更新
date: 2018-06-30 11:11:11.000000000 +09:00
---
## 前言
Apollo 在 v0.10.0 版本后，支持自动更新。v0.10.0之前的版本在配置变化后不会重新注入，需要重启才会更新。

也就是说，如果一个属性加入了 @Value 注解，并且这个配置在配置中心也存在，那么，配置中心修改属性值后，就会自动更新这个值。同时，有个开关可以控制这个功能是否关闭（默认开启)。
配置文件中写入 `apollo.autoUpdateInjectedSpringProperties = false` 即可关闭该功能。

## 如何实现？

此次提交的 PR 详情可见  https://codecov.io/gh/ctripcorp/apollo/pull/972/diff.

大致先说下实现思路：

Apollo 实现了` BeanPostProcessor` 接口，这个接口的作用则是`每个 bean 初始化成前后做操作`。

在 `postProcessBeforeInitialization` 方法中，会取出这个 Bean 的所有属性和方法，并判断他们是否含有 `@Value` 注解从而进行**处理**。

具体处理逻辑，则是: 将符合条件的属性封装成一个  `SpringValue` 对象，放在一个` Map` 中。当 `clien` 检测到配置发生变化时，就会更新这个 `Map` 里面的值，从而达到自动更新的目的。

当然，这只是大概的思路，具体细节则要复杂一些。接下来我们就说说具体的实现细节。

## 实现细节

**相关类**：

1. `SpringValue` @Value 注解的详细信息数据结构。
2. `SpringValueRegistry` @Value 注册中心，保存了他的 key/value 机构。
2. `SpringValueDefinitionProcessor` 针对 Spring 3.x 版本做的特殊操作。
3. `SpringValueProcessor` 处理 @Value 注解的类。
4. `ConfigPropertySourcesProcessor` 注册 SpringValueProcessor 到容器。
5. `PropertySourcesProcessor` 将 Config 和自动更新监听器绑定，同时注入 Spring 环境。


**逻辑步骤**：
1. 无论是 XML 方式，注解方式，SpringBoot 方式，都会触发注册机制，即自动注册 `SpringValueProcessor`  处理器到 Spring 容器（他主要是个 `BeanPostProcessor`）。这是 apollo 自定义的 `@Value` 处理器。

2. 不仅仅注册 `SpringValueProcessor` ，还注册 `PropertySourcesProcessor`，这是一个 `BeanFactoryPostProcessor`，即在 Spring bean 工厂初始化后，可以进行修改的一个类。这里的时机比 `SpringValueProcessor` 早。

3. `PropertySourcesProcessor` 在具体方法中，会初始化所有的 Config(`ConfigService.getConfig(namespace)`)，并设置到 Spring 的环境中（存储所有的 Property，且同名状态下优先级最高，目的是让 Spring 自己注入到变量中）。同时，创建一个自动更新监听器，监听所有的 Config。

4. `SpringValueProcessor`  在容器初始化 Bean 的时候，会处理所有带有 `@Value` 注解的类，并放入到 `SpringValueRegistry` 的 Map 中。注意：`SpringValueRegistry` 是单例的。而 自动更新监听器 也是包含一个 `SpringValueRegistry` 的。因此，每当一个 Config 变化的时候，都会触发 change 事件，并调用监听器的 onChange 方法，如果匹配，该方法则会更新 `SpringValueRegistry`  内部的值——完成自动更新。

5. 这里有个问题：所有的配置都放在 `SpringValueRegistry` 中，一个 client 会持有多个 `namespace`，每个 `namespace` 可能会有重名的配置。那么会不会发生冲突呢？实际上，apollo 考虑到了这点，在设计 `namespace` 的时候，就有一个 order 属性，用于处理这种情况，**order 越小，优先级越高**。当一个配置没有显时的设置 `namespace` 时，apollo 将其归纳为优先级最高的 `namespace`——即 order 最小的 `namespace`。而实现优先级的则是 `Spring PropertySource` 内部的排序机制，说白了就是一个 List，优先级最高的下标为0，当循环匹配的时候，优先匹配下标为0的配置。

6. 在 `shouldTriggerAutoUpdate` 方法里，有个判断很绕：`根据 key 获取 Spring 环境中的配置值，判断这个值和刚刚发生的变化值是否相等，如果相等，就更新 value，反之，跳过此次事件`。
解释一下：**我们知道，`@value` 对应的是优先级最高的 `namespace`，`environment `获取的也是优先级最高的 namespace 的配置。**
如果一个配置更新了，但 `environment` 优先级最高的配置却没有更新，那么 `@value` 对应的更新事件就不应该触发。
如果一个配置更新了，`environment` 获取到的优先级最高的配置也更新了，那么 `@value` 对应的更新时间就应该触发。
这里，其实就是 `@value` 是否更新的重要判断。



## 总结

关于这个小功能，为什么要单独拉出来说一说呢？实际上，该功能涉及到的东西还是很多的。例如：
1. 一个普通的 Java 项目如何和 Spring 框架融合 —— 注册，初始化，注入 Spring 环境等操作。
2. 如何玩转 Spring 的 `PropertySource` 的 order 机制。
3. 每一个 `namespace` 都有一个长轮询，发生更新后，apollo 触发监听器的更新事件，其中包括`自动更新监听器`，但是，需要通过 `shouldTriggerAutoUpdate` 的判断才能进行更新，因为 @value 可能会和多个 namespace 重名，需要通过优先级来过滤。即：如果 Spring 环境中优先级最高的 config 更新了，那么 `@value` 对应的 field 就需要更新，反之，不能更新（`namespace` 不匹配）。


















