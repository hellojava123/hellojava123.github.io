---
layout: post
title: Raft 配置变更 Configuration changes 拾遗
date: 2018-10-26 05:01:01.000000000
---
## 背景

仔细思考了 Raft 关于配置变更的内容，发现论文中的搞法用代码根本无法实现.......然后开始了 search 的阶段，翻到知乎一篇文章和作者的博士论文，感觉用这个可以实现配置变更。

在这里稍微记录一下。

## 新的发现

知乎作者 孙建良 的一篇 《Raft 一致性协议》 专栏里，提到了 Raft 作者的长达 240 页的博士论文里，关于配置变更的详细实现。

首先，要解决的问题：分布式集群机器的增减。

新的发现是什么呢？

答：每次只向集群添加一个节点。

Raft phd 34 页：

![](https://upload-images.jianshu.io/upload_images/4236553-59e3a463884d9eac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先解释一下这幅图：蓝色方框里显示 old 配置的大多数，红色方框里显示 new 配置的大多数。我们以 b 为例，b 中，向原本 3 个节点的集群添加一个节点变成 4 个节点（暂时不考虑 2n + 1 问题），那么，如果 leader 崩溃，无论是新的配置，还是旧的配置，肯定存在节点交集。在 b 中，old 配置想赢得选举，必须有 2 个节点支持，new 配置要想赢的选举，必须有 3 个节点支持（新配置是 4 个节点），那么他们必然就有一个相交的节点。

这将带来什么影响？

任何一方想赢得选举，都必须争取这个节点的选票。换句话说，同一时刻，有且只有一个 leader 产生。无论是新的，还是旧的。这样就解决了之前那篇文章提到的“出现  2 个 leader 的问题”。

然后我们再假设一下：

我们有一个集群，现在有 3 个节点，然后我们添加一个节点，并更新了 leader 的配置为 4 节点，然后把复制到其他 2 个节点，这时，leader 出现了崩溃，重新选举。

这个时候，会有 2 个结果：
1. 新配置复制到了集群的大多数（大多数的值在这里必须大于 2 （包括 leaer 自身））。
    * 如果新配置复制到了大多数集群，那么新 leader 肯定使用的是新的配置。
2. 新配置没有复制到集群的大多数。
    * 如果新配置没有复制到大多数集群，那么新 leader 肯定使用的是老的配置。


## 代码如何实现？

通过上面的分析，实现起来就比较简单了。

思路：
1. 每次只增加一个节点，如果要增加 2 个节点，必须等上次那个节点添加成功，才能继续添加。否则会出现双 leader 的情况。
2. 添加节点时，新节点使用的自然是新的配置。
3. 添加的第一步，是否应该是将新节点的日志和 leader 进行同步？如果同步，那么新节点将有可能成为 leader，如果不同步，新节点只能是 follower。
4. 第二步，leader 将新的配置把自身先更新，然后并行的发送到其他 follower。等待反馈，如果大部分节点复制成功，那么，leader ，新的配置就成功了。


意外：

如果 leader 复制的过程中出现了崩溃，那么就重新开始选举，此时，一共有 4 个节点，可以确定的是：leader + 新节点都是 new 配置，而，另外两个的配置不一定。关键就在于这两个节点的配置，如果有一个是新的配置，那么选举出来的，肯定是 new 配置的 leader。反之，肯定是 old 配置的 leader。

如果选举出来的是 new 配置节点，那么需要将这个集群的配置刷新，即在此将配置重新发送到所有 follower。

如果选举出来的是 old 配置节点，那么，old 集群 leader 也照样并行的刷新他所在的  3 个节点（包括自己）。 新的节点直接忽略。

在客户端，如果添加节点失败，则进行重试。



## 参考 
[英文 paper  pdf 地址](https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf)


[Raft paper 中文翻译 —— 寻找一种易于理解的一致性算法（扩展版）](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)


[Raft 作者讲解视频](https://www.youtube.com/watch?v=YbZ3zDzDnrw&feature=youtu.be)


[Raft 作者讲解视频对应的 PPT](http://www2.cs.uh.edu/~paris/6360/PowerPoint/Raft.ppt)


[一个简单的讲解 Raft 协议的动画](http://thesecretlivesofdata.com/raft/)
 

[Raft 一致性协议------知乎 孙建良](https://zhuanlan.zhihu.com/p/29678067)


[别再怀疑自己的智商了，Raft协议本来就不好理解 ----- 老钱](https://zhuanlan.zhihu.com/p/36547283)

[Raft 作者 240 页的博士论文](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)

