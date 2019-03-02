---
layout: post
title: apollo 10 — adminService 全量发布
date: 2018-07-19 00:01:01.000000000 +09:00
---
## 目录

1. UI 界面
2. Portal 服务
3. admin 服务
4. 总结

## 1. UI 界面

![1](https://upload-images.jianshu.io/upload_images/4236553-c928a28afb3b6f24.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![2](https://upload-images.jianshu.io/upload_images/4236553-f965473b05543d8a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![3](https://upload-images.jianshu.io/upload_images/4236553-11704e0d594f7c79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2. Portal 服务

当我们点击上面的发布按钮的时候，调用的当然是 portal 的接口。具体代码如下：

```java
  /**
   * 全量发布
   * @param appId SampleApp
   * @param env DEV
   * @param clusterName default
   * @param namespaceName  application
   * @param branchName 分支/灰度名称
   * @param deleteBranch true
   * @param model {"releaseTitle":"20180716220550-gray-release-merge-to-master","releaseComment":"","isEmergencyPublish":false}
   * @return
   */
  @PreAuthorize(value = "@permissionValidator.hasReleaseNamespacePermission(#appId, #namespaceName)")
  @RequestMapping(value = "/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/branches/{branchName}/merge", method = RequestMethod.POST)
  public ReleaseDTO merge(@PathVariable String appId, @PathVariable String env,
                          @PathVariable String clusterName, @PathVariable String namespaceName,
                          @PathVariable String branchName, @RequestParam(value = "deleteBranch", defaultValue = "true") boolean deleteBranch,
                          @RequestBody NamespaceReleaseModel model) {
    // 如果是紧急发布,但该环境不允许紧急发布,抛出异常
    if (model.isEmergencyPublish() && !portalConfig.isEmergencyPublishAllowed(Env.fromString(env))) {
      throw new BadRequestException(String.format("Env: %s is not supported emergency publish now", env));
    }
    // 合并主版本和灰度版本, 得到一个发布 dto
    ReleaseDTO createdRelease = namespaceBranchService.merge(appId, Env.valueOf(env), clusterName, namespaceName, branchName,
                                                             model.getReleaseTitle(), model.getReleaseComment(),
                                                             model.isEmergencyPublish(), deleteBranch);

    ConfigPublishEvent event = ConfigPublishEvent.instance();
    event.withAppId(appId)
        .withCluster(clusterName)
        .withNamespace(namespaceName)
        .withReleaseId(createdRelease.getId())
        .setMergeEvent(true)
        .setEnv(Env.valueOf(env));

    publisher.publishEvent(event);// 发送邮件

    return createdRelease;
  }
```

接口职责不多：是否符合紧急发布的数据校验，调用 Service， 发布“配置发布”事件（发送邮件）。


看看调用 Service 的过程，该方法称为 merge ，实际上就是合并灰度和主版本的配置。代码如下：

```java
  public ReleaseDTO merge(String appId, Env env, String clusterName, String namespaceName,
                          String branchName, String title, String comment,
                          boolean isEmergencyPublish, boolean deleteBranch) {
    // 计算 changeSets
    ItemChangeSets changeSets = calculateBranchChangeSet(appId, env, clusterName, namespaceName, branchName);
    // 调用 admin 服务
    ReleaseDTO mergedResult =
        releaseService.updateAndPublish(appId, env, clusterName, namespaceName, title, comment,
                                        branchName, isEmergencyPublish, deleteBranch, changeSets);

    Tracer.logEvent(TracerEventType.MERGE_GRAY_RELEASE,
                 String.format("%s+%s+%s+%s", appId, env, clusterName, namespaceName));

    return mergedResult;
  }
```
做了 2 件事情： 计算 change 集合，调用 admin 服务。很明显，计算 change 对于 protal 非常重要。

calculateBranchChangeSet 方法主要将灰度配置和主版本配置合并。

代码：

```java
private ItemChangeSets calculateBranchChangeSet(String appId, Env env, String clusterName, String namespaceName,
                                                String branchName) {
  NamespaceBO parentNamespace = namespaceService.loadNamespaceBO(appId, env, clusterName, namespaceName);// 父版本 namespace

  if (parentNamespace == null) {
    throw new BadRequestException("base namespace not existed");
  }

  if (parentNamespace.getItemModifiedCnt() > 0) {
    throw new BadRequestException("Merge operation failed. Because master has modified items");
  }

  List<ItemDTO> masterItems = itemService.findItems(appId, env, clusterName, namespaceName);// 主版本 items 

  List<ItemDTO> branchItems = itemService.findItems(appId, env, branchName, namespaceName);// 子版本 items 

  ItemChangeSets changeSets = itemsComparator.compareIgnoreBlankAndCommentItem(parentNamespace.getBaseInfo().getId(),
                                                                               masterItems, branchItems);// 得到 changeSet
  changeSets.setDeleteItems(Collections.emptyList());// 防止误删除，emm，灰度的内容并不是全量的，因此上面的计算有些问题，并且目前没有删除功能。所以这里可以置空。
  changeSets.setDataChangeLastModifiedBy(userInfoHolder.getUser().getUserId());
  return changeSets;
}
```

步骤：
1. 获取主版本的 namespace 详细信息，用于数据检验，id 赋值。
2. 获取主版本的所有 item 配置，再获取灰度版本的所有 item 配置，注意，灰度版本的 item 只有其自身新增的和修改的配置，不是全量的（这将导致后面一个奇怪的现象）。
3. 比较两者差异，得到 change 集合。
4. 设置 deleteList 为空 —— 奇怪现象（`灰度的内容并不是全量的，因此上面的计算有些问题，并且目前没有删除功能。所以这里可以置空, 并且防止误删除`）。
5. 设置修改人。

这里需要注意的是计算差异到底是怎么计算的，为什么后面有置空 deleteItem 的操作。

我就不贴全部的方法了，贴一下对删除操作有影响的代码：

```java
/** 比较,忽略空格,返回一个改变的 items */
public ItemChangeSets compareIgnoreBlankAndCommentItem(long baseNamespaceId, List<ItemDTO> baseItems, List<ItemDTO> targetItems){

  // 忽略新增/修改 item 代码......

  // 处理删除,但这个逻辑似乎不对. 不过此类不知道数据来源,工具类没有问题.
  for (ItemDTO item: baseItems){// 主版本
    String key = item.getKey();

    ItemDTO targetItem = targetItemMap.get(key);
    if(targetItem == null){//delete// 如果灰度版本里没有,说明删除了.
      changeSets.addDeleteItem(item);// 添加进删除集合
    }
  }
  return changeSets;
}
```

可以看到，这段代码里，循环主版本，逐个对比灰度版本，如果灰度版本里没有，就添加进 delete 集合，而我们知道，灰度版本的 item 只有修改的和新增的，这时，将导致误删除。

**但这个工具类的计算是没有问题的，有问题的是外层数据的完整性。**

因此需要在外面打个补丁：`changeSets.setDeleteItems(Collections.emptyList());`


好，计算完 changeSet，就要调用 admin 服务了，并且把 changeSet 传递过去，然后返回一个 release 对象，表示发布成功，并发布事件。

在分析 admin 之前，总结一下 protal 的流程：

![](https://upload-images.jianshu.io/upload_images/4236553-bd30447caecf0a76.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)









## 3. admin 服务

从 portal 的代码中，可以看到，调用的是 admin 的 updateAndPublish 方法接口，看看这个接口：
位置 ： `com.ctrip.framework.apollo.adminservice.controller.ReleaseController.java`
代码如下：

```java
  @Transactional
  @RequestMapping(path = "/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/updateAndPublish", method = RequestMethod.POST)
  public ReleaseDTO updateAndPublish(@PathVariable("appId") String appId,// 应用名称
                                     @PathVariable("clusterName") String clusterName,//集群
                                     @PathVariable("namespaceName") String namespaceName,// 主版本名称
                                     @RequestParam("releaseName") String releaseName, // 发布名称
                                     @RequestParam("branchName") String branchName,// 灰度名称 cluster
                                     @RequestParam(value = "deleteBranch", defaultValue = "true") boolean deleteBranch,// 是否删除灰度
                                     @RequestParam(name = "releaseComment", required = false) String releaseComment,// 评论
                                     @RequestParam(name = "isEmergencyPublish", defaultValue = "false") boolean isEmergencyPublish,// 是否紧急发布
                                     @RequestBody ItemChangeSets changeSets) {// 这个是 portal 发来的
    Namespace namespace = namespaceService.findOne(appId, clusterName, namespaceName);// 找到分支
    if (namespace == null) {
      throw new NotFoundException(String.format("Could not find namespace for %s %s %s", appId,
                                                clusterName, namespaceName));
    }
    // 合并改变 并且发布
    Release release = releaseService.mergeBranchChangeSetsAndRelease(namespace, branchName, releaseName,
                                                                     releaseComment, isEmergencyPublish, changeSets);
    // 是否删除分支
    if (deleteBranch) {
      namespaceBranchService.deleteBranch(appId, clusterName, namespaceName, branchName,
                                          NamespaceBranchStatus.MERGED, changeSets.getDataChangeLastModifiedBy());
    }
    // 保存发布消息到数据库
    messageSender.sendMessage(ReleaseMessageKeyGenerator.generate(appId, clusterName, namespaceName),
                              Topics.APOLLO_RELEASE_TOPIC);

    return BeanUtils.transfrom(ReleaseDTO.class, release);
  }
```

这个接口接受 portal 调用，比较有趣的点是，这里的 changeSet 是 portal 计算的，而不是 admin 自己计算的。

然后，controller 层比较简单，数据校验，调用 Service，发送消息。

当然主要看看 Service。

主要是 releaseService 的 mergeBranchChangeSetsAndRelease 方法，看名字，任务很多：合并分支修改集合，并且发布。


代码如下：

```java
@Transactional
public Release mergeBranchChangeSetsAndRelease(Namespace namespace, String branchName, String releaseName,
                                               String releaseComment, boolean isEmergencyPublish,
                                               ItemChangeSets changeSets) {
  // 检查锁
  checkLock(namespace, isEmergencyPublish, changeSets.getDataChangeLastModifiedBy());
  /// 更新 item
  itemSetService.updateSet(namespace, changeSets);
  // 找到最新发布的 release
  Release branchRelease = findLatestActiveRelease(namespace.getAppId(), branchName, namespace
      .getNamespaceName());
  // release Id
  long branchReleaseId = branchRelease == null ? 0 : branchRelease.getId();
  // 找到当前 namespace 的所有 Item(刚刚更新的)
  Map<String, String> operateNamespaceItems = getNamespaceItems(namespace);

  Map<String, Object> operationContext = Maps.newHashMap();
  // 构造操作上下文 sourceBranch=灰度名称 baseReleaseId=最新的releaseId isEmergencyPublish=是否紧急发布, 用于构建发布历史
  operationContext.put(ReleaseOperationContext.SOURCE_BRANCH, branchName);
  operationContext.put(ReleaseOperationContext.BASE_RELEASE_ID, branchReleaseId);
  operationContext.put(ReleaseOperationContext.IS_EMERGENCY_PUBLISH, isEmergencyPublish);
  // ReleaseHistory Audit 主版本
  return masterRelease(namespace, releaseName, releaseComment, operateNamespaceItems,
                       changeSets.getDataChangeLastModifiedBy(),
                       // 灰度合并回主分支发布
                       ReleaseOperation.GRAY_RELEASE_MERGE_TO_MASTER, operationContext);

}
```

代码很简单，步骤：
1. 检查锁，和普通发布一样，判断修改者和发布者是不是同一个人。
2. 根据 Portal 传递来的 changeSets 更新 item。
3. 找到最新发布的 release（构建发布历史的上下文）。
4. 发布主版本。

其中，updateSet 方法比较重要，要看看他是怎么更新 item 的。

方法很长，总之，就是将 changeSet 的内容保存到主版本的 namespace 下。

```java
@Transactional
public ItemChangeSets updateSet(String appId, String clusterName,
                                String namespaceName, ItemChangeSets changeSet) {
  //  最后改变数据的人
  String operator = changeSet.getDataChangeLastModifiedBy();
  // 改变数据的详细信息
  ConfigChangeContentBuilder configChangeContentBuilder = new ConfigChangeContentBuilder();
  // 如果创建了新的
  if (!CollectionUtils.isEmpty(changeSet.getCreateItems())) {
    // 循环
    for (ItemDTO item : changeSet.getCreateItems()) {
      // 转换
      Item entity = BeanUtils.transfrom(Item.class, item);
      entity.setDataChangeCreatedBy(operator);
      entity.setDataChangeLastModifiedBy(operator);
      // 保存 item 到数据库
      Item createdItem = itemService.save(entity);
      // 保存到 builder createItems List 中
      configChangeContentBuilder.createItem(createdItem);
    }
    // 最后记录审核
    auditService.audit("ItemSet", null, Audit.OP.INSERT, operator);
  }
  // 如果有修改的数据
  if (!CollectionUtils.isEmpty(changeSet.getUpdateItems())) {
    for (ItemDTO item : changeSet.getUpdateItems()) {
      // 转换并寻找
      Item entity = BeanUtils.transfrom(Item.class, item);
      Item managedItem = itemService.findOne(entity.getId());
      // 不存在抛出异常
      if (managedItem == null) {
        throw new NotFoundException(String.format("item not found.(key=%s)", entity.getKey()));
      }
      // 之前的数据
      Item beforeUpdateItem = BeanUtils.transfrom(Item.class, managedItem);

      //protect. only value,comment,lastModifiedBy,lineNum can be modified
      // 将之前数据内容更新
      managedItem.setValue(entity.getValue());
      managedItem.setComment(entity.getComment());
      managedItem.setLineNum(entity.getLineNum());
      managedItem.setDataChangeLastModifiedBy(operator);
      // 更新
      Item updatedItem = itemService.update(managedItem);
      // 更新 builder 中 value
      configChangeContentBuilder.updateItem(beforeUpdateItem, updatedItem);

    }
    // 最后审核 itemSet
    auditService.audit("ItemSet", null, Audit.OP.UPDATE, operator);
  }
  // 如果有删除的
  if (!CollectionUtils.isEmpty(changeSet.getDeleteItems())) {
    for (ItemDTO item : changeSet.getDeleteItems()) {
      // 数据库删除
      Item deletedItem = itemService.delete(item.getId(), operator);
      // 添加到 builder 中
      configChangeContentBuilder.deleteItem(deletedItem);
    }
    // 审核
    auditService.audit("ItemSet", null, Audit.OP.DELETE, operator);
  }
  // 如果 builder 中有内容
  if (configChangeContentBuilder.hasContent()){
    // 创建提交记录
    createCommit(appId, clusterName, namespaceName,
        configChangeContentBuilder.build(), // 将 build 变成 json 保存
                 changeSet.getDataChangeLastModifiedBy());
  }

  return changeSet;
}
```

在成功更新 itme 之后，便可以进行最终的发布了，发布很简单，就不展开讲了。

然后看看删除灰度，默认是要删除的。

步骤：
1. 找到灰度发布的最新  release。
2. 更新灰度规则，置空灰度规则。
3. 删除灰度 cluster 和关联的 namespace。置于灰度为什么和 cluster 关联，而不是和 namespace 关联，这是因为最初的 apollo 没有设计灰度，后面加上灰度的时候，为了避免 namespace 大幅修改，就在 cluster 里加入父子逻辑了（咨询过作者）。
4. 记录发布历史。根据是否 merge 记录是放弃灰度还是合并后删除，方便审计。

发布操作有很多类型，apollo 的常量如下：

```java
public interface ReleaseOperation {
  int NORMAL_RELEASE = 0;//普通发布
  int ROLLBACK = 1;// 回滚
  int GRAY_RELEASE = 2;// 灰度发布
  int APPLY_GRAY_RULES = 3;// 灰度规则更新
  int GRAY_RELEASE_MERGE_TO_MASTER = 4;// 灰度合并回主分支发布
  int MASTER_NORMAL_RELEASE_MERGE_TO_GRAY = 5;// 主分支发布灰度自动发布
  int MATER_ROLLBACK_MERGE_TO_GRAY = 6;// 主分支回滚灰度自动发布
  int ABANDON_GRAY_RELEASE = 7;//放弃灰度
  int GRAY_RELEASE_DELETED_AFTER_MERGE = 8;// 灰度版本合并后删除
}
```

总结一下 admin 的发布流程：

![](https://upload-images.jianshu.io/upload_images/4236553-44ee282e64da1d29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 4. 总结

将 portal 和 admin 组合起来看，下图：

![](https://upload-images.jianshu.io/upload_images/4236553-3aef348a7726dacb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

























