---
layout: post
title: Apollo 8 — ConfigService 异步轮询接口的实现
date: 2018-07-18 00:01:01.000000000 +09:00
---
##  源码

Apollo 长轮询的实现，是通过客户端轮询 `/notifications/v2` 接口实现的。具体代码在 com.ctrip.framework.apollo.configservice.controller.NotificationControllerV2.java。

这个类也是实现了 ReleaseMessageListener 监控，表明他是一个消息监听器，当有新的消息时，就会调用他的 hanlderMessage 方法。这个具体我们后面再说。

该类只有一个 rest 接口： pollNotification 方法。返回值是 DeferredResult，这是 Spring 支持 Servlet 3 的一个类，关于异步同步的不同，可以看笔者的另一篇文章 [异步 Servlet 和同步 Servlet 的性能测试](http://thinkinjava.cn/2018/07/%E5%BC%82%E6%AD%A5-Servlet-%E5%92%8C%E5%90%8C%E6%AD%A5-Servlet-%E7%9A%84%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95/)。


该接口提供了几个参数：
1. appId appId
2. cluster 集群名称
3. notificationsAsString 通知对象的 json 字符串
4. dataCenter，idc 属性
5. clientIp 客户端 IP， 非必传，为了扩展吧估计

大家有么有觉得少了什么？ namespace 。

当然，没有 namespace 这个重要的参数是不存在的。

参数在 notificationsAsString 中。客户端会将自己所有的  namespace 传递到服务端进行查询。

是时候上源码了。

```java
@RequestMapping(method = RequestMethod.GET)
  public DeferredResult<ResponseEntity<List<ApolloConfigNotification>>> pollNotification(
      @RequestParam(value = "appId") String appId,// appId
      @RequestParam(value = "cluster") String cluster,// default
      @RequestParam(value = "notifications") String notificationsAsString,// json 对象 List<ApolloConfigNotification>
      @RequestParam(value = "dataCenter", required = false) String dataCenter,// 基本用不上, idc 属性
      @RequestParam(value = "ip", required = false) String clientIp) {

    List<ApolloConfigNotification> notifications =// 转换成对象
          gson.fromJson(notificationsAsString, notificationsTypeReference);
          
    // Spring 的异步对象: timeout 60s, 返回304
    DeferredResultWrapper deferredResultWrapper = new DeferredResultWrapper();
    Set<String> namespaces = Sets.newHashSet();
    Map<String, Long> clientSideNotifications = Maps.newHashMap();
    Map<String, ApolloConfigNotification> filteredNotifications = filterNotifications(appId, notifications);// 过滤一下名字
    // 循环
    for (Map.Entry<String, ApolloConfigNotification> notificationEntry : filteredNotifications.entrySet()) {
      // 拿出 key
      String normalizedNamespace = notificationEntry.getKey();
      // 拿出 value
      ApolloConfigNotification notification = notificationEntry.getValue();
      /* 添加到 namespaces Set */
      namespaces.add(normalizedNamespace);
      // 添加到 client 端的通知, key 是 namespace, values 是 messageId
      clientSideNotifications.put(normalizedNamespace, notification.getNotificationId());
      // 如果不相等, 记录客户端名字
      if (!Objects.equals(notification.getNamespaceName(), normalizedNamespace)) {
        // 记录 key = 标准名字, value = 客户端名字
        deferredResultWrapper.recordNamespaceNameNormalizedResult(notification.getNamespaceName(), normalizedNamespace);
      }
    }// 记在 namespaces 集合, clientSideNotifications 也put (namespace, notificationId)

    // 组装得到需要观察的 key,包括公共的.
    Multimap<String, String> watchedKeysMap =
        watchKeysUtil.assembleAllWatchKeys(appId, cluster, namespaces, dataCenter);// namespaces 是集合
    // 得到 value; 这个 value 也就是 appId + cluster + namespace
    Set<String> watchedKeys = Sets.newHashSet(watchedKeysMap.values());
    // 从缓存得到最新的发布消息
    List<ReleaseMessage> latestReleaseMessages =// 根据 key 从缓存得到最新发布的消息.
        releaseMessageService.findLatestReleaseMessagesGroupByMessages(watchedKeys);

    /* 如果不关闭, 这个请求将会一直持有一个数据库连接. 影响并发能力. 这是一个 hack 操作*/
    entityManagerUtil.closeEntityManager();
    // 计算出新的通知
    List<ApolloConfigNotification> newNotifications =
        getApolloConfigNotifications(namespaces, clientSideNotifications, watchedKeysMap,
            latestReleaseMessages);
    // 不是空, 理解返回结果, 不等待
    if (!CollectionUtils.isEmpty(newNotifications)) {
      deferredResultWrapper.setResult(newNotifications);
    } else {
      // 设置 timeout 回调:打印日志
      deferredResultWrapper
          .onTimeout(() -> logWatchedKeys(watchedKeys, "Apollo.LongPoll.TimeOutKeys"));
      // 设置完成回调:删除 key
      deferredResultWrapper.onCompletion(() -> {
        //取消注册
        for (String key : watchedKeys) {
          deferredResults.remove(key, deferredResultWrapper);
        }
      });

      //register all keys 注册
      for (String key : watchedKeys) {
        this.deferredResults.put(key, deferredResultWrapper);
      }
    }
    // 立即返回
    return deferredResultWrapper.getResult();/** @see DeferredResultHandler 是关键 */
  }
```

注释写了很多了，再简单说说逻辑：

1. 解析 JSON 字符串为 List< ApolloConfigNotification> 对象。
2. 创建 Spring 异步对象。
3. 处理过滤 namespace。
4. 根据 namespace 生成需要监听的 key，格式为 appId + cluster + namespace，包括公共 namespace。并获取最新的 Release 信息。
5. 关闭 Spring 实例管理器，释放数据库资源。
6. 根据刚刚得到的 ReleaseMessage，和客户端的 ReleaseMessage 的版本进行对比，生成新的配置通知对象集合。
7. 如果不是空 —— 立即返回给客户端，结束此次调用。如果没有，进入第 8 步。
8. 设置 timeout 回调方法 —— 打印日志。再设置完成回调方法：删除注册的 key。
9. 对客户端感兴趣的 key 进行注册，这些 key 都对应着 deferredResultWrapper 对象，可以认为他就是客户端。
10. 返回 Spring 异步对象。该请求将被异步挂起。

Apollo 的 DeferredResultWrapper 保证了 Spring 的 DeferredResult 对象，泛型内容是 List<ApolloConfigNotification>， 构造这个对象，默认的 timeout 是 60 秒，即挂起 60 秒。同时，对 setResult 方法进行包装，加入了对客户端 key 和服务端 key 的一个映射（大小写不一致） 。

我们刚刚说，Apollo 会将这些 key 注册起来。那么什么时候使用呢，异步对象被挂起，又是上面时候被唤醒呢？

答案就在 handleMessage 方法里。我们刚刚说他是一个监听器，当消息扫描器扫描到新的消息时，会通知所有的监听器，也就是执行 handlerMessage 方法。方法内容如下：

```java
@Override
public void handleMessage(ReleaseMessage message, String channel) {

  String content = message.getMessage();
  if (!Topics.APOLLO_RELEASE_TOPIC.equals(channel) || Strings.isNullOrEmpty(content)) {
    return;
  }
  String changedNamespace = retrieveNamespaceFromReleaseMessage.apply(content);

  //create a new list to avoid ConcurrentModificationException 构造一个新 list ,防止并发失败
  List<DeferredResultWrapper> results = Lists.newArrayList(deferredResults.get(content));

  // 创建通知对象
  ApolloConfigNotification configNotification = new ApolloConfigNotification(changedNamespace, message.getId());
  configNotification.addMessage(content, message.getId());

  //do async notification if too many clients 如果有大量的客户端(100)在等待,使用线程池异步处理
  if (results.size() > bizConfig.releaseMessageNotificationBatch()) {
    // 大量通知批量处理
    largeNotificationBatchExecutorService.submit(() -> {
      for (int i = 0; i < results.size(); i++) { // 循环
        /*
         * 假设一个公共 Namespace 有10W 台机器使用，如果该公共 Namespace 发布时直接下发配置更新消息的话，
         * 就会导致这 10W 台机器一下子都来请求配置，这动静就有点大了，而且对 Config Service 的压力也会比较大。
         * 即"惊群效应"
         */
        if (i > 0 && i % bizConfig.releaseMessageNotificationBatch() == 0) {// 如果处理了一批客户端,休息一下(100ms)
            TimeUnit.MILLISECONDS.sleep(bizConfig.releaseMessageNotificationBatchIntervalInMilli());
        }
        results.get(i).setResult(configNotification);// 通知每个等待的 HTTP 请求
      }
    });
    return;
  }

  // 否则,同步处理
  for (DeferredResultWrapper result : results) {
    result.setResult(configNotification);
  }
}
```

笔者去除了一些日志和一些数据判断。大致的逻辑如下：
1. 消息类型必须是 “apollo-release”。然后拿到消息里的 namespace 内容。
2. **根据 namespace 从注册器里拿出 Spring 异步对象集合**。
3. 创建通知对象。
4. 如果有超过 100 个客户端在等待，那么就使用线程池批量执行通知。否则就同步慢慢执行。
5. 每处理 100 个客户端就休息 100ms，防止发生惊群效应，导致大量客户端调用配置获取接口，引起服务抖动。
6. 循环调用 Spring 异步对象的 setResult 方法，让其立即返回。

具体的流程图如下：


![](https://upload-images.jianshu.io/upload_images/4236553-ca1bbe21c2a21b52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中，灰色区域是扫描器的异步线程，黄色区域是接口的同步线程。他们共享 deferredResults 这个线程安全的 Map，实现异步解耦和实时通知客户端。

## 总结

好了，这就是 Apollo 的长轮询接口，客户端会不断的轮询服务器，服务器会 Hold住  60 秒，这是通过 Servlet 3 的异步 + NIO 来实现的，能够保持万级连接（Tomcat 默认 10000）。

通过一个线程安全的 Map + 监听器，让扫描器线程和 HTTP 线程共享 Spring 异步对象，即实现了消息实时通知，也让应用程序实现异步解耦。


























































