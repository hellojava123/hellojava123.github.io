---
layout: post
title: Apollo 6 — ConfigService 获取配置接口
date: 2018-07-16 00:01:01.000000000 +09:00
---
## 大纲

看本文之前，建议看看 apollo 的官方文档，特别是数据库设计文档。

1. 主流程分析

2.1 聊聊细节

2.2 loadConfig() 加载配置

2.3 auditReleases() 方法记录此次访问详情



## 1. 主流程分析

具体代码在 `com.ctrip.framework.apollo.configservice.controller.ConfigController#queryConfig` 方法中。

代码如下：

```java
// 这个 . 号是防止 Spring 框架去除了 . 号后面的字符,例如 xxx.json, xxx.properties
@RequestMapping(value = "/{appId}/{clusterName}/{namespace:.+}", method = RequestMethod.GET)
public ApolloConfig queryConfig(@PathVariable String appId, @PathVariable String clusterName,
                                @PathVariable String namespace,
                                @RequestParam(value = "dataCenter", required = false) String dataCenter,
                                @RequestParam(value = "releaseKey", defaultValue = "-1") String clientSideReleaseKey,//20180704093033-648d208dc9c1c9be
                                @RequestParam(value = "ip", required = false) String clientIp,
                                @RequestParam(value = "messages", required = false) String messagesAsString,//{"details":{"SampleApp+default+application":19}}
                                HttpServletRequest request, HttpServletResponse response) throws IOException {
  String originalNamespace = namespace;
  //strip out .properties suffix 剔除后缀
  namespace = namespaceUtil.filterNamespaceName(namespace);
  //fix the character case issue, such as FX.apollo <-> fx.apollo
  // 改名字
  namespace = namespaceUtil.normalizeNamespace(appId, namespace);

  if (Strings.isNullOrEmpty(clientIp)) {
    clientIp = tryToGetClientIp(request);// 获取客户端 IP
  }

  // 转换成对象（目前没有地方用到？）
  ApolloNotificationMessages clientMessages = transformMessages(messagesAsString);

  List<Release> releases = Lists.newLinkedList();

  String appClusterNameLoaded = clusterName;
  // appID 如果不是占位符号
  if (!ConfigConsts.NO_APPID_PLACEHOLDER.equalsIgnoreCase(appId)) {
    // 获取发布信息
    Release currentAppRelease = configService.loadConfig(appId, clientIp, appId, clusterName, namespace,
        dataCenter, clientMessages);
    if (currentAppRelease != null) {
      releases.add(currentAppRelease);// 添加进集合
      //we have cluster search process, so the cluster name might be overridden 这个解释看不懂??? 集群搜索过程指的是?
      appClusterNameLoaded = currentAppRelease.getClusterName();// 使用release 的集群名称.
    }
  }
  // 关联类型?
  //if namespace does not belong to this appId, should check if there is a public configuration
  // 如果命名空间不属于这个 appId, 那么应该检查他是否是公共配置
  if (!namespaceBelongsToAppId(appId, namespace)) {
    Release publicRelease = this.findPublicConfig(appId, clientIp, clusterName, namespace,
        dataCenter, clientMessages);// 获取公共配置
    if (!Objects.isNull(publicRelease)) {
      releases.add(publicRelease);// 添加进集合
    }
  }

  if (releases.isEmpty()) {// 空的话,返回 404
    response.sendError(HttpServletResponse.SC_NOT_FOUND,//404
        String.format(
            "Could not load configurations with appId: %s, clusterName: %s, namespace: %s",
            appId, clusterName, originalNamespace));
    Tracer.logEvent("Apollo.Config.NotFound",
        assembleKey(appId, clusterName, originalNamespace, dataCenter));
    return null;
  }

  auditReleases(appId, clusterName, dataCenter, clientIp, releases);// 保存实例信息,(就是配置灰度规则时的ip选择列表+ 实例列表)
  // + 号拼接所有的 release key
  String mergedReleaseKey = releases.stream().map(Release::getReleaseKey)
          .collect(Collectors.joining(ConfigConsts.CLUSTER_NAMESPACE_SEPARATOR));

  if (mergedReleaseKey.equals(clientSideReleaseKey)) {// 如果客户端那边的 key 和这边的 key 一致,则 304
    // Client side configuration is the same with server side, return 304
    response.setStatus(HttpServletResponse.SC_NOT_MODIFIED);// 返回 304
    Tracer.logEvent("Apollo.Config.NotModified",
        assembleKey(appId, appClusterNameLoaded, originalNamespace, dataCenter));
    return null;
  }
  // releaseKey 不一致,则创建 config 对象返回.
  ApolloConfig apolloConfig = new ApolloConfig(appId, appClusterNameLoaded, originalNamespace,
      mergedReleaseKey);
  apolloConfig.setConfigurations(mergeReleaseConfigurations(releases));// 合并所有的配置, 其中,私有配置优先公共配置

  Tracer.logEvent("Apollo.Config.Found", assembleKey(appId, appClusterNameLoaded,
      originalNamespace, dataCenter));
  return apolloConfig;
}
```

代码有点长，具体细节等下慢慢聊，这里说说主要逻辑：
1. 调整 namespace 的名字。获取客户端 IP（为了灰度）。

2. 判断 appId 是不是占位符。如果不是，就尝试加载该 AppId 下的 Cluster 下的 namespace 的 release 配置。并添加进结果集。

3. 判断是否是公共 namespac， 假设这个 namespace 不属于当前 AppId，那么就是公共配置，需要加载公共配置（通常就是管理的 namespace）。**注意：这个时候，可能会有 2 个结果集：当前 AppId 发布的`重写公共配置的配置 `+ 公共配置。**

4. 如果结果集合是空，返回 404。

5. auditReleases 方法会异步的保存此次客户端获取配置的详细信息到数据库中，portal 页面就可以看到这些信息了。

6. 比较服务端的 key 和客户端的 key 是否相同，因为每次发布配置都会有一个唯一的 key 生成，这里比较一下，就可以知道配置是否发生更改，如果相同，返回  304.

7. 如果不同，构造一个 Config 对象返回给客户端。这里有个注意的地方: `mergeReleaseConfigurations` 方法会将 release 集合反转一下，目的是让私有的重写配置优先于公共的配置。


目前来看，不是很复杂，主要就是根据指定的 namespace 加载配置，并和客户端的 key 进行比较。如果不同，就返回新的配置。

## 2.1 聊聊细节

步骤1，2 都是处理 namespace，大小写，后缀什么的，优先使用服务端的名称。

转换了 messagesAsString 为 ApolloNotificationMessages 对象，目前没用到。

可以看到，比较重要的方法就是 configService.loadConfig 方法。这个方法是获取配置的核心方法。下面的获取公共配置的 findPublicConfig 方法内部也是调用的此方法。

然后还有 auditReleases 方法，这个其实就是记录此次客户端获取配置的详细信息的。

然后还有一个反转方法。这个很简单，大家可以自己看看。

先看看 configService.loadConfig  方法。

## 2.2 loadConfig() 加载配置

说方法之前，先看看 ConfigService 这个接口。最上层的是监听器接口，用于监听消息变化。然后是 ConfigService 接口，定义 loadConfig 方法并返回 release 对象。

最下面是具体实现类，抽象类是用了模板模式，定义了获取配置的骨架，下面则有 2 个实现类，一个基于缓存，一个基于 DB。默认是 DB。具体使用哪个类要看 server_config 表里的配置，配置的 key 是 ` config-service.cache.enabled`，value 要么 true 要么 false。

注意，使用缓存很耗费内存，小心 OOM 哦。

![](https://upload-images.jianshu.io/upload_images/4236553-91250356549325a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看看抽象类里 loadConfig 方法的实现：

```java
  @Override
  public Release loadConfig(String clientAppId, String clientIp, String configAppId, String configClusterName,
      String configNamespace, String dataCenter, ApolloNotificationMessages clientMessages) {
    // load from specified cluster fist  如果不是默认的 cluster(私有或者灰度)
    if (!Objects.equals(ConfigConsts.CLUSTER_NAME_DEFAULT, configClusterName)) {
      Release clusterRelease = findRelease(clientAppId, clientIp, configAppId, configClusterName, configNamespace,
          clientMessages);

      if (!Objects.isNull(clusterRelease)) {
        return clusterRelease;
      }
    }

    // try to load via data center 试图通过数据中心获取
    if (!Strings.isNullOrEmpty(dataCenter) && !Objects.equals(dataCenter, configClusterName)) {
      Release dataCenterRelease = findRelease(clientAppId, clientIp, configAppId, dataCenter, configNamespace,
          clientMessages);
      if (!Objects.isNull(dataCenterRelease)) {
        return dataCenterRelease;
      }
    }

    // fallback to default release 从默认的 cluster 获取
    return findRelease(clientAppId, clientIp, configAppId, ConfigConsts.CLUSTER_NAME_DEFAULT, configNamespace,
        clientMessages);
  }
```

步骤：首先加载私有/灰度的 cluster，这个就是客户端配置文件里的 `apollo.cluster` 配置， 然后再加载 `server.properties` 配置文件的 idc 属性，最后加载默认的。

他们的优先级如图：

![图片来自 apollo wiki](https://upload-images.jianshu.io/upload_images/4236553-b220502cf8731b27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

细心的你可以发现，3 个 if 判断力都是调用的 findRelease 方法，只是第四个参数不同，这个参数就是 configClusterName —— 不同的 cluster。

这个方法首先或加载灰度的，然后再加载普通的。所以，客户端的 IP 就显得重要了。

```java
  private Release findRelease(String clientAppId, String clientIp, String configAppId, String configClusterName,
      String configNamespace, ApolloNotificationMessages clientMessages) {
    // 获取灰度release id, 从应用的缓存中获取规则.
    Long grayReleaseId = grayReleaseRulesHolder.findReleaseIdFromGrayReleaseRule(clientAppId, clientIp, configAppId,
        configClusterName, configNamespace);

    Release release = null;

    if (grayReleaseId != null) {// 如果有灰度
//      return releaseRepository.findByIdAndIsAbandonedFalse(releaseId);// 没有放有回滚的发布
      release = findActiveOne(grayReleaseId, clientMessages); // 获取灰度 release
    }

    if (release == null) {// 如果没有发布的新灰度
      // 获取最新的普通的发布活动
      release = findLatestActiveRelease(configAppId, configClusterName, configNamespace, clientMessages);
    }
    // 灰度 --> 最新的
    return release;
  }
````

步骤：调用` grayReleaseRulesHolder `的`findReleaseIdFromGrayReleaseRule ` 方法获取灰度发布 ID。当然，这也是个缓存。

如果有灰度，则根据 id 获取对应的 release 信息，得到配置，release 里面包含了全量的配置信息（从数据库获取）。

如果没有灰度，则获取最新的普通的发布信息（从数据库获取）。

关键在于获取灰度 id。

看看这个方法：

```java
  public Long findReleaseIdFromGrayReleaseRule(String clientAppId, String clientIp, String
      configAppId, String configCluster, String configNamespaceName) {
    String key = assembleGrayReleaseRuleKey(configAppId, configCluster, configNamespaceName);// 组装 key
    if (!grayReleaseRuleCache.containsKey(key)) {// 从缓存中获取, 缓存是个 handler 监听器 + 定时任务
      return null;
    }
    //create a new list to avoid ConcurrentModificationException 如果存在,就处理
    List<GrayReleaseRuleCache> rules = Lists.newArrayList(grayReleaseRuleCache.get(key));
    for (GrayReleaseRuleCache rule : rules) {
      //check branch status 必须是激活的状态
      if (rule.getBranchStatus() != NamespaceBranchStatus.ACTIVE) {
        continue;
      }// 如果匹配上了 ip 和 客户端的 appId, 就返回
      if (rule.matches(clientAppId, clientIp)) {
        return rule.getReleaseId();
      }
    }
    return null;
  }
```

步骤：首先用 + 号将 appId，cluster，namespace 拼接，再从缓存中获取，获取的是该 key 对应的灰度规则。

而缓存由一个定时任务更新（60s） + 监听器更新。

如果存在，就循环比较规则，如果规则是激活状态，且 appId 和 ip 和当前客户端匹配，那么就返回这个灰度的 release Id。

**这个就是得到灰度 release Id 的具体逻辑，可以看到，这里是优先加载灰度的。**

那么这个定时任务 + 监听器具体是怎么样的呢？

刚刚说了，定时任务是 60 s 一次，相对于配置中心来说，及时性肯定是不够的，所以，他更多的是一种补偿措施，即监听器失效了，定时任务能够保证 60s 内配置是最新的。

而监听器才是最新的配置。具体方法则是 handleMessage 方法。这个方法会得到一个发布消息，包含 appId + clusterName + namespace，有了这个信息，就可以得到 release 信息了。

每当发布一个配置 ，或回滚一个配置，都会发送一个消息到数据库，ConfigService 会扫描得到这个消息，然后通知所有的监听器。执行监听器的 handlerMessage 方法。

在灰度规则监听器中，会检查灰度发布规则表，根据消息的内容（`appId + cluster + namespace 组成的唯一 namespace key`）并进行处理，处理的逻辑则是更新缓存中的规则内容。

**总结一下这个 loadConfig 方法：**
> 这是 ConfigService 接口定义的方法，由 一个抽象类和 2 个派生类组成，默认使用 DB 模式的派生类，抽象类定义了 loadConfig 的方法骨架，利用模板模式，2 个子类可以根据自己的特性返回数据 —— release。loadConfig 里，在获取 release 的时候，会有一个查找顺序，首先找私有/灰度的 cluster，然后找 idc（这个一般公司用不到，携程内部的特性），最后找默认的。而他们调用的 findRelease 方法内部也会有一个查找顺序：首先根据灰度规则查找灰度发布 ID，如果没有，查找默认的最新发布 —— 也就是灰度的规则比默认的规则高。


## 2.3 auditReleases() 方法记录此次访问详情

这个方法主要是调用 instanceConfigAuditUtil 的 audit 方法：

```java  
  private void auditReleases(String appId, String cluster, String dataCenter, String clientIp,
                             List<Release> releases) {
    if (Strings.isNullOrEmpty(clientIp)) {
      //no need to audit instance config when there is no ip
      return;
    }
    for (Release release : releases) {
      instanceConfigAuditUtil.audit(appId, cluster, dataCenter, clientIp, release.getAppId(),
          release.getClusterName(),
          release.getNamespaceName(), release.getReleaseKey());
    }
  }
```

注意：这里的判断，如果没有 ip， 就不记录了。这里用的是 aduit 审计的概念，我想就是类似记录吧，方便后面进行复盘，查看啥的。

而这个 audit 方法具体的内容则是：构造一个 InstanceConfigAuditModel 对象放到一个阻塞队列中，由另一个线程异步处理。

```java
public boolean audit(String appId, String clusterName, String dataCenter, String
      ip, String configAppId, String configClusterName, String configNamespace, String releaseKey) {
    return this.audits.offer(new InstanceConfigAuditModel(appId, clusterName, dataCenter, ip,
        configAppId, configClusterName, configNamespace, releaseKey));
  }
// 长度为 10000 的阻塞队列
private BlockingQueue<InstanceConfigAuditModel> audits = Queues.newLinkedBlockingQueue
      (10000);
```
那么异步执行的内容是怎么样的呢？

当然要看队列 poll 或者 take 后做什么了.

```java
  @Override
  public void afterPropertiesSet() throws Exception {
    auditExecutorService.submit(() -> {
      while (!auditStopped.get() && !Thread.currentThread().isInterrupted()) {
        try {
          InstanceConfigAuditModel model = audits.poll();
          if (model == null) {
            TimeUnit.SECONDS.sleep(1);
            continue;
          }
          doAudit(model);
        } catch (Throwable ex) {
          Tracer.logError(ex);
        }
      }
    });
  }
```

该类实现了 Spring 的 InitializingBean 接口，重写了 afterPropertiesSet 方法，这个方法会在属性注入完毕后执行。

方法其实就是提交了一个任务，任务内容则是从队列中取出对象，然后执行 doAudit 方法。如果取出是空，休眠 1 秒。

这个方法的主要内容就是更新客户端访问信息，或者创建客户端访问信息。使用 2 个缓存，存储 instanceId 和 releaseKey，这个是为了提高校验数据的性能。

而这个实例在数据库是这样的：

![Instance 表](https://upload-images.jianshu.io/upload_images/4236553-d4b127c8616a00d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![InstanceConfig 表](https://upload-images.jianshu.io/upload_images/4236553-a9756f415935b821.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从 Instance 表结构看，记录是每台机器最新访问的记录，而 InstanceConfig 则是记录的此次访问的具体 namespace 的 发布信息。

对应的是控制台的实例列表：

![](https://upload-images.jianshu.io/upload_images/4236553-80cb0841855353b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/4236553-6be410c980026a7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



关于这个方法的具体内容我就不贴了，感兴趣的可以自己看看，主要内容就是记录 Instance 的访问信息用于后台审计查看。


## 总结

好了，apollo 客户端访问 ConfigService 获取配置的大概思路和具体细节就介绍完了。这里总结一下。

1. 获取配置的时候，可能会有 2 个结果集（关联类型），那么会将私有的优先（放到前面）。如果集合是空，返回 404 ，如果没有新的发布信息，返回 304.

2. 当服务器加载配置信息的时候，有几个顺序，特别是集群的顺序：私有/灰度 cluster （apollo.cluster）----> 数据中心（server.properties 的 idc）-----> 默认的 cluster。同时，加载集群内部配置的时候，也会优先加载灰度的配置（根据 IP），然后才是默认的配置。

3. 最后，会记录此次访问的信息，方便后台审计。如果是 10 分钟之内访问的，即不会更新。


































