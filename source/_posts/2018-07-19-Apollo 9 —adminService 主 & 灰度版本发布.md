---
layout: post
title: Apollo 9 — adminService 主 & 灰度版本发布
date: 2018-07-19 00:01:01.000000000 +09:00
---
## 目录
1. Controller 层
2. Service 层 publish 方法
3. 发送 ReleaseMessage 消息
4. 总结

## 1. Controller 层

主版本发布即点击主版本发布按钮：

![](https://upload-images.jianshu.io/upload_images/4236553-c827c10ed25ebd7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体接口位置：`com.ctrip.framework.apollo.adminservice.controller` 包下 `ReleaseController#publish`
实际上灰度版本发布也是调用这个接口的。
代码：

```java
  /**
   * 主版本发布
   */
  @Transactional
  @RequestMapping(path = "/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/releases", method = RequestMethod.POST)
  public ReleaseDTO publish(@PathVariable("appId") String appId,
                            @PathVariable("clusterName") String clusterName,
                            @PathVariable("namespaceName") String namespaceName,
                            @RequestParam("name") String releaseName,
                            @RequestParam(name = "comment", required = false) String releaseComment,
                            @RequestParam("operator") String operator,
                            @RequestParam(name = "isEmergencyPublish", defaultValue = "false") boolean isEmergencyPublish) {
    // 校验存在与否
    Namespace namespace = namespaceService.findOne(appId, clusterName, namespaceName);
    if (namespace == null) {
      throw new NotFoundException(String.format("Could not find namespace for %s %s %s", appId,
                                                clusterName, namespaceName));
    }
    // 发布
    Release release = releaseService.publish(namespace, releaseName, releaseComment, operator, isEmergencyPublish);

    //send release message 发送消息到 ReleaseMessage
    Namespace parentNamespace = namespaceService.findParentNamespace(namespace);
    String messageCluster;
    if (parentNamespace != null) {
      messageCluster = parentNamespace.getClusterName();
    } else {
      messageCluster = clusterName;
    }
    messageSender.sendMessage(ReleaseMessageKeyGenerator.generate(appId, messageCluster, namespaceName),
                              Topics.APOLLO_RELEASE_TOPIC);
    return BeanUtils.transfrom(ReleaseDTO.class, release);
  }
```

该层主要做了 2 件事情，1是调用 Service 层的 public 方法做真正的发布操作，2是发送“发布消息”到数据库——等待 ConfigService 消费。

所以，我们主要关注 Service 层的 publish 方法。

## 2. Service 层 publish 方法

该方法有些繁琐，主要流程图如下：

![ publish 流程图](https://upload-images.jianshu.io/upload_images/4236553-7e6e131876893357.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以通过比对流程图和代码来看。

代码如下：

```java
  @Transactional
  public Release publish(Namespace namespace, String releaseName, String releaseComment,
                         String operator, boolean isEmergencyPublish) {
    // 检查锁
    checkLock(namespace, isEmergencyPublish, operator);
    // 获取 item
    Map<String, String> operateNamespaceItems = getNamespaceItems(namespace);

    // 根据当前 namespace 找到父 namespace, 也就是灰度的主版本.
    Namespace parentNamespace = namespaceService.findParentNamespace(namespace);

    //branch release // 父 namespace 不是 null, 说明当前就是灰度版本.
    if (parentNamespace != null) {
      // 发布灰度版本.
      return publishBranchNamespace(parentNamespace, namespace, operateNamespaceItems,
                                    releaseName, releaseComment, operator, isEmergencyPublish);
    }

    // 非灰度版本, 找到子版本
    Namespace childNamespace = namespaceService.findChildNamespace(namespace);

    Release previousRelease = null;
    if (childNamespace != null) {
      // 找到上一个版本
      previousRelease = findLatestActiveRelease(namespace);
    }

    //master release
    Map<String, Object> operationContext = Maps.newHashMap();
    // 记录是否紧急发布
    operationContext.put(ReleaseOperationContext.IS_EMERGENCY_PUBLISH, isEmergencyPublish);
    // 主版本发布
    Release release = masterRelease(namespace, releaseName, releaseComment, operateNamespaceItems,
                                    operator, ReleaseOperation.NORMAL_RELEASE, operationContext);


    //merge to branch and auto release
    // 将主版本合并到灰度版本. 并自动发布
    if (childNamespace != null) {
      mergeFromMasterAndPublishBranch(namespace, childNamespace, operateNamespaceItems,
                                      releaseName, releaseComment, operator, previousRelease,
                                      release, isEmergencyPublish);
    }
    return release;
  }
```

1. 检查锁：如果不是紧急发布，就需要检查锁，如果这个 namespace 的最后修改者就是当前用户，那么就抛出异常。禁止其修改。

2. 根据 namespace 获取所有的 item，也就是配置。

3. 判断当前的 namespace 是否有父 namespace，如果有，说明当前 namespace 是灰度 namespace，则进行灰度发布（主版本发布和灰度发布逻辑不同）。

这里说下父子 namespace 在 apollo 的设计：

![主体E-R Diagram  图片来自 apollo wiki](https://upload-images.jianshu.io/upload_images/4236553-e94fb9c5c13105d7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中可以看出，namespace 和 cluster 是多对一的关系，而 cluster 有个字段：ParentClusterId，也就是说，cluster 是有层级的。每当创建一个灰度配置，实际上，就是创建了一个新的 cluster，这个新的 cluster 的名字就是 `时间戳-字符串`，大概是这样的：`20180705150428-1dc5208dc9e8146b`. 然后再在这个新 cluster 下面创建新的 namespace，那么，namespace 无形中也有了层级（父子）关系。

4. 如果没有父 namespace，说明是主版本发布，那么就需要处理他的子 （灰度）版本，同时，为了后面比对灰度版本和上一个版本的区别（如果灰度修改了上一个版本的数据，就需要记录，否则，灰度数据和主版本将无法对应），还要记录上一个版本的 release 信息。

5. **发布主版本**。并保存发布历史。

6. 如果存在灰度版本，就更新灰度版本的配置，并发布灰度版本。

关于灰度版本，这里多提一句，每次发布都是一个 release，release 对象有个 configuration，包含了此次发布的全量配置，因此，灰度发布的 configuration 中，包含了每次对应的主版本的配置，如果主版本发生了变化，那么灰度版本肯定也是要变更的。所以需要重新发布灰度版本。

其中关键的方法就是 `mergeConfiguration`，该方法表明了灰度发布的主要逻辑：

```java
  private Map<String, String> mergeConfiguration(Map<String, String> baseConfigurations,
                                                 Map<String, String> coverConfigurations) {
    Map<String, String> result = new HashMap<>();
    //copy base configuration
    for (Map.Entry<String, String> entry : baseConfigurations.entrySet()) {
      result.put(entry.getKey(), entry.getValue());
    }

    //update and publish
    for (Map.Entry<String, String> entry : coverConfigurations.entrySet()) {
      result.put(entry.getKey(), entry.getValue());
    }

    return result;
  }
```

方法很简单：两个参数，主版本配置，灰度版本配置。首先将主版本配置保存到 Map 中，然后将灰度版本配置也 put 到 Map 中，利用 Map 唯一 Key 的特性，保证灰度版本覆盖主版本。

所以这个方法的 put 顺序决定了**灰度版本覆盖主版本。**

publish 方法更多的细节不再赘述，有疑惑的地方可以交流。

## 3. 发送 ReleaseMessage 消息

这个发送消息的操作本来应该是 MQ，apollo 为了减少依赖，直接使用的 mysql，但已经留好了MQ 的设计。关于 ReleaseMessage 的设计，我这里引用一下 apollo 的文档：

> Admin Service在配置发布后，需要通知所有的Config Service有配置发布，从而Config Service可以通知对应的客户端来拉取最新的配置。
从概念上来看，这是一个典型的消息使用场景，Admin Service作为producer发出消息，各个Config Service作为consumer消费消息。通过一个消息组件（Message Queue）就能很好的实现Admin Service和Config Service的解耦。
>在实现上，考虑到Apollo的实际使用场景，以及为了尽可能减少外部依赖，我们没有采用外部的消息中间件，而是通过数据库实现了一个简单的消息队列。
>**实现方式如下：**
>1.  Admin Service在配置发布后会往ReleaseMessage表插入一条消息记录，消息内容就是配置发布的AppId+Cluster+Namespace，参见[DatabaseMessageSender](https://github.com/ctripcorp/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/DatabaseMessageSender.java)
>2.  Config Service有一个线程会每秒扫描一次ReleaseMessage表，看看是否有新的消息记录，参见[ReleaseMessageScanner](https://github.com/ctripcorp/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/ReleaseMessageScanner.java)
>3.  Config Service如果发现有新的消息记录，那么就会通知到所有的消息监听器（[ReleaseMessageListener](https://github.com/ctripcorp/apollo/blob/master/apollo-biz/src/main/java/com/ctrip/framework/apollo/biz/message/ReleaseMessageListener.java)），如[NotificationControllerV2](https://github.com/ctripcorp/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/controller/NotificationControllerV2.java)，消息监听器的注册过程参见[ConfigServiceAutoConfiguration](https://github.com/ctripcorp/apollo/blob/master/apollo-configservice/src/main/java/com/ctrip/framework/apollo/configservice/ConfigServiceAutoConfiguration.java)
>4.  NotificationControllerV2得到配置发布的AppId+Cluster+Namespace后，会通知对应的客户端

示意图如下：

![](https://upload-images.jianshu.io/upload_images/4236553-e1372f1232717d10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


apollo 定义了 MessageSender 接口，定义了一个 sendMessage 方法，这个方法目前只有基于 Mysql 的实现，即 DatabaseMessageSender 实现类。

该类会将数据直接保存到数据库。然后清理掉`比刚刚存的消息旧的消息`—— 防止消息表不断增大。

## 4. 总结

发布分为主版本发布，灰度版本发布，全量发布，这次说了前两个，全量发布下次再说。

而主/灰发布的一个比较繁琐的地方就是两个版本的合并，**灰度版本发布要合并主版本。主版本发布要更新灰度版本**。

同时，灰度的设计也有点绕，中间隔了一层 cluster。


在发布成功之后，需要发送消息到数据库，让 ConfigService 能够感知到此次发布，并通知客户端。关于如何通知客户端，下次再说。



































































