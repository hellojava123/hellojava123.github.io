---
layout: post
title: Raft 日志复制 Log replication
date: 2018-10-26 00:01:03.000000000
---
## 背景

Raft 是分布式一致性算法，保证的实际上是多台机器上数据的一致性，前面说的 leader 选举是为了保证日志复制的一致性。

简单来说，保证复制日志相同，才是分布式一致性算法的最终任务。

Leader 选举只是为了保证日志复制相同的辅助工作。实际上，在更为学术的 Paxos 里面，是没有 leader 的概念的（大部分 Paxos 的实现通常会加入 leader 机制提高性能）。

**所以，保证复制日志相同，就是分布式一致性算法的终极任务**。

## 日志复制

在 Raft 中，leader 会接收客户端的所有需求（follower 会将写请求转发给 leader），leader 会将数据以日志的方式通过 RPC 的方式同步给所有 followers，只要超过半数以上的 follower 反馈成功，这条日志就成功提交了。如果 RPC 请求超时，leader 就不停的进行 RPC 重试。

下面再从几个方面说说日志复制：
1. 复制过程
2. 日志的组成
3. 主从日志的一致性
4.  日志特性
5. 日志的不正常情况


##### 复制过程
1. 客户端的每一个请求都包含被复制状态机执行的指令。
2. leader 把这个指令作为一条新的日志条目添加到日志中，然后并行发起 RPC 给其他的服务器，让他们复制这条信息。
3. 假如这条日志被安全的复制，领导人就应用这条日志到自己的状态机中，并返回给客户端。
4. 如果 follower 宕机或者运行缓慢或者丢包，领导人会不断的重试，知道所有的 follower 最终都存储了所有的日志条目。

大概的流程如下图：

![图 1](https://upload-images.jianshu.io/upload_images/4236553-c9ad172e3f41ea02.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





#####  日志的组成

日志的数据结构：

1. 创建日志时的任期号（用来检查节点日志是否出现不一致的情况）
2. 状态机需要执行的指令（真正的内容）
3. 索引：整数索引表示日志条目在日志中位置

日志结构如下图：

![图 2](https://upload-images.jianshu.io/upload_images/4236553-466b0d0790cd1e8e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图显示，共有 8 条日志，提交了  7 条。提交的日志都将通过状态机持久化到磁盘中，防止宕机。


#####  主从日志的一致性

然后谈谈主从日志的一致性问题，这个是分布式一致性算法要解决的根本问题。

Raft 为了保证主从日志的一致性，做了以下规则/限制（补丁）。

1. Raft 保证所有已提交的日志条目都是持久化的并且最终会被所有可用的状态机执行
2. 领导人把指令作为一条新的日志条目添加到日志中后，将并行的发送 RPC 给 follower，让他们复制这条信息。
3. leader 将会跟踪最大的且即将被提交的日志条目的索引，并且这个索引会被包含在未来的所有附加日志 RPC 请求中，这样就能保证其他的服务器知道 leader 的索引提交位置。
4. 一旦 follower 知道一条日志条目已经被提交，那么他也会将这条日志应用到自己的状态机中（按照日志的顺序）。


#####  日志特性

1. 如果在不同的日志中的两个日志条目的`索引` 和 `索引下标` 相同，那么他们的指令就是相同的。（`原因：leader
最多在一个任期里的一个日志索引位置创建一条日志条目，日志条目在日志的位置从来不会改变`）

2. 如果在不同的日志里的 2 个日志条目拥有相同的任期号和索引，那么他们之前的日志项都是相同的。（`原因：每次 RPC 发送附加日志时，leader 会把这条日志条目的前面的日志的下标和任期号一起发送给 follower，如果 follower 发现和自己的日志不匹配，那么就拒绝接受这条日志，这个称之为一致性检查`）。
   2.1 这里需要提一下 Raft 的日志匹配规则：`如果 2 个日志的相同的索引位置的日志条目的任期号相同，那么 Raft 就认为这个日志从头到这个索引之间全部相同` ，这个非常重要。

##### 日志的不正常情况

上面说的都是日志在正常情况下的表现，没有考虑到一些异常情况，例如 leader 崩溃。

即，正常情况下， leader 和 follower 的日志保持一致性，所以附加日志 RPC 的一致性检查从来不会失败（查询上次已提交的日志条目的任期和下标）

然而，让我们考虑一下 leader 的崩溃：假设老的 leader 还没有完全复制完所有的日志条目，就崩溃了，这将导致 follower 的日志有可能比 leader 的日志多，也可能少，也可能多多少少。。。。

下图将展示 leader 和 follower 的日志的冲突情况：

![图 3](https://upload-images.jianshu.io/upload_images/4236553-8a3893a9355685c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


从上图可以看出，所有的 follower 都和 leader 的日志冲突了，leader 的最后一条日志的任期是 6， 下标是 10 ，而其他 follower 的日志都与 leader 不匹配。

如何处理？

Raft 给出了一个方案（补丁）:通过强制 follower 直接复制 leader 的日志解决（意味着 follower 中的和 leader 冲突的日志将被覆盖）。

如何操作？

要使得 follower 的日志和 leader 进入一致状态，leader 必须找到 follower 最后一条和 leader 匹配的日志，然后把那条日志后面的日志全部删除。

依据这个限制，上图中的 a follower 不需要删除任何条目，b 也不需要删除，c follower 需要删除最后一个条目，d follower 需要删除最后 2 个任期为 7 的条目，e 需要删除最后 2 个任期为 4 的条目，f 则比较厉害，需要删除 下标为 3 之后的所有条目。

Raft 如何实现？

leader 为每一个 follower 维护一个下标，称之为 nextIndex，表示下一个需要发送给 follower 的日志条目的索引。

当一个新 leader 刚获得权力的时候，他将自己的最后一条日志的 index + 1，也就是上面提的 nextIndex 值，如果一个 follower 的日志和 leader 不一致，那么在下一次  RPC 附加日志请求中，一致性检查就会失败（不会插入数据）。

当这种情况发生，leader 就会把 nextIndex 递减进行重试，直到遇到匹配到正确的日志。

当匹配成功之后，follower 就会把冲突的日志全部删除，此时，follower 和 leader 的日志就达成一致。

## 日志复制 Summary

日志复制是分布式一致性算法的核心，所谓的一致性，就是集群多节点的数据一致性。

Raft 把每条日志都附加了 任期号和下标 来保证日志的唯一性。

依据这个限制，Raft 对日志有以下保证：如果 2 个日志的相同的索引位置的日志条目的任期号相同，那么 Raft 就认为这个日志从头到这个索引之间全部相同。


依据这个保证，当 leader 和 follower 日志冲突的时候，leader 将校验 follower 最后一条日志是否和 leader 匹配，如果不匹配，将递减查询，直到匹配，匹配后，删除冲突的日志。这样就实现了主从日志的一致性。


## 参考 
[英文 paper  pdf 地址](https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf)

[Raft paper 中文翻译 —— 寻找一种易于理解的一致性算法（扩展版）](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)

[Raft 作者讲解视频](https://www.youtube.com/watch?v=YbZ3zDzDnrw&feature=youtu.be)

[Raft 作者讲解视频对应的 PPT](http://www2.cs.uh.edu/~paris/6360/PowerPoint/Raft.ppt)

[一个简单的讲解 Raft 协议的动画](http://thesecretlivesofdata.com/raft/)









