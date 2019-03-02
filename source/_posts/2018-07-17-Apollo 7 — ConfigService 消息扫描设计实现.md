---
layout: post
title: Apollo 7 — ConfigService 消息扫描设计实现
date: 2018-07-17 00:01:01.000000000 +09:00
---
## 目录

1. 设计
2. 代码实现
3. 总结

## 1.设计

Apollo 为了减少依赖，将本来 MQ 的职责转移到了 Mysql 中。具体表现为 Mysql  中的 ReleaseMessage 表。

具体官方文档可见：[发送ReleaseMessage的实现方式](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1#211-%E5%8F%91%E9%80%81releasemessage%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F)

用张图简单的来表示一下 ：

![](https://upload-images.jianshu.io/upload_images/4236553-6d2573985f45307c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有人肯定要问了，为什么 Admin Service 和 Config Service 不放在一起呢？我曾提过 issue 问过作者，大概的答案是：两者职责不同，部署的实例数也不同，通常 Admin 会少一些，因为只服务于 Portal，而 Config 则要部署的多一些，因为需要服务于 Client。

第二则是两者的开发节奏也是不一样，Config Service 的更新会影响客户端，而 Admin 的更新则是影响 Portal，所以，分开他们，对服务的部署可以更加细粒度。

关于这个 issue：[为什么 admin 和 config 不合在一起呢 ](https://github.com/ctripcorp/apollo/issues/1204)

高可用的话，可参考下图或链接 [可用性考虑](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E8%AE%BE%E8%AE%A1#%E5%9B%9B%E5%8F%AF%E7%94%A8%E6%80%A7%E8%80%83%E8%99%91):
![](https://upload-images.jianshu.io/upload_images/4236553-90043d3f9fdc395b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

扯远了。

回到我们之前说的，之所以要使用 Mysql，是因为要减少依赖，为了替代 MQ，而之所以要使用 MQ，是为了解耦 Config 和 Admin，而之所以要使用 Config 和 Admin，则是因为设计，部署，开发节奏等原因。

那么，基于 Mysql 的消息的实现是怎么弄的呢？

![](https://upload-images.jianshu.io/upload_images/4236553-c1122096ce2191a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面是 apollo 文档对于`发送ReleaseMessage的实现方式`的描述。

我们来看看代码实现. 

## 2. 代码实现

在 `com.ctrip.framework.apollo.biz.message` 包下，有关于消息的实现，

![](https://upload-images.jianshu.io/upload_images/4236553-da2a33db3c377dda.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

麻雀虽小，五脏俱全。

1. MessageSender 接口定义了一个方法：`void sendMessage(String message, String channel);`
这个 channel 就是 topic 了。 message 就是消息的具体内容了。

2. 目前只有一个实现类 `DatabaseMessageSender`， 基于数据库的消息发送。就是把消息保存到 ReleaseMessage 表中。

3. Topics 定义消息主题，目前只有一个：`apollo-release`。

4. ReleaseMessageListener 消息监听器接口，定义了一个方法：`void handleMessage(ReleaseMessage message, String channel)`，需要实现此方法并注册到扫描器中，当有新的消息时，便会通知监听器。

5. ReleaseMessageScanner 消息扫描器，用于扫描数据库的 ReleaseMessage 表，如果有新数据，则通知监听器。

目前监听器有以下实现：

![](https://upload-images.jianshu.io/upload_images/4236553-ef4f3d5f64d33763.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从左到右：
1. ReleaseMessageServiceWithCache，ReleaseMessage 的缓存，用于长轮询判断是否以后新的消息。
2. GrayReleaseRulesHolder，灰度规则变化监听器。
3. ConfigService，分为缓存和默认，默认的 handleMessage 方法什么都不做，缓存实现则会对 cache 热身。
4. NotificationControllerV2 监听器，当客户端被长连接 Hold 住时，消息如果更新则唤醒客户端立即返回，保证及时性。
5. NotificationController 已废弃，不谈了。

所以，每当 ReleaseMessageScanner 得到新的消息，都会触发这些监听器。这些监听器在哪里被添加的呢？

位置:`com.ctrip.framework.apollo.configservice.ConfigServiceAutoConfiguration.java`

![](https://upload-images.jianshu.io/upload_images/4236553-75508923db204cb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ReleaseMessageScanner 的 afterPropertiesSet 方法会启动一个间隔 1 秒定时任务，执行 scanMessages 方法。

具体方法如下：

```java
private boolean scanAndSendMessages() {
    //current batch is 500 批处理 500 条
    // 根据 maxIdScanned 找到比这个 id 大的 500 条数据,
    List<ReleaseMessage> releaseMessages =
        releaseMessageRepository.findFirst500ByIdGreaterThanOrderByIdAsc(maxIdScanned);
    if (CollectionUtils.isEmpty(releaseMessages)) {
      return false;
    }
    // 开始通知 handleMessage 监听器
    fireMessageScanned(releaseMessages);
    int messageScanned = releaseMessages.size();// 消息数量
    maxIdScanned = releaseMessages.get(messageScanned - 1).getId();// 更新最大 id
    
    return messageScanned == 500;// 如果不足 500, 说明没有新消息了
  }
```

方法很简单，首先根据最大的扫描 Id  找到 500 条消息，对这 500 条消息进行批处理，触发监听器。最后更新最大扫描 Id。如果此次取出的数据量超过 500 条，则认为还有数据，就继续处理。

很明显，每次处理这么多消息肯定很耗时，那么假设处理这些消息要 5 秒，那么定时任务的间隔是多少了呢？答：6 秒。因为定时任务的模式是 scheduleWithFixedDelay 模式，固定的间隔，以当前任务结束时间 + period 时间作为下一个任务的开始时间。


那么，admin 什么时候会向数据库发送消息（保存 ReleaseMessage）呢?

那么就要查看 sendMessage 方法被哪些地方调用就好了。

![](https://upload-images.jianshu.io/upload_images/4236553-e29490700794ebc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. NamespaceBranchController
  1.1 更新灰度规则 `updateBranchGrayRules`
  1.2 删除灰度分支 `deleteBranch`

2. NamespaceService
  2.1 删除命名空间 ` NamespaceController#deleteNamespace`
  2.2 删除集群 `ClusterController#delete`
  2.3 删除灰度 `NamespaceBranchController#deleteBranch `
  2.4 删除命名空间分支` NamespaceService#deleteNamespace  `

3. ReleaseController
  3.1 主/灰版本发布 `publish`
  3.2 全量发布 `updateAndPublish`
  3.3 回滚 `rollback `

这些地方被触发的时候，都会发送消息到数据库。因此，可能会重复触发，导致 ConfigService 重复消费。。。例如在放弃灰度和全量发布的时候，就会重复发送消息。

笔者就这个问题提了 issue，期待官方 fix 这个 bug。


## 3. 总结

本文重点分析了 apollo 关系消息这块的设计： 每当有 app， cluster，namespace， Release 等操作的时候，都会发送消息到数据库，ConfigService 会定时扫描数据库，有新消息了，就会立即通知各个监听器，确保配置实时推送到客户端。

同时，我们也发现了一个 bug，就是会重复发送消息，引起重复消费。

迫于种种限制，apollo 使用了这种设计，获取，如果可以使用 MQ 的话，一切会更加的简单。













































