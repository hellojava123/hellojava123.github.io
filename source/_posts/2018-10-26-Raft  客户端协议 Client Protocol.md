---
layout: post
title: Raft  客户端协议 Client Protocol
date: 2018-10-26 00:03:01.000000000
---
## 摘要

名词：
线性化操作：每一次操作立即执行，在 TA 调用和收到回复之间，只执行一次。

## 基本操作

1. 客户端启动时，随机挑选一台服务器。
   1.1 如过第一次挑选的不是 leader， 那么那个服务器会拒绝客户端的请求，并提供最近接收到的 leader 的信息（ip+port）；
    1.2 如果 leader 奔溃了，客户端请求超时。
   
2. 客户端进行重试。

如图 ：

![](https://upload-images.jianshu.io/upload_images/4236553-0840e29122537393.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***

## 安全性

1.  Raft 需要实现线性化语义，但是，上述操作没有考虑 leader 奔溃的情况。

例如，leader 在提交这条日志后，在响应客户端**之前** 奔溃了，那么客户端会和新的 leader 重试这个指令，导致重复执行。

解决方案，其实和互联网的幂等操作相同：客户端给每一个指令指定一个唯一序列号，服务器状态机跟踪这个序列号，当服务器接收到这个指令时，首先校验序列号 —— 防止重复执行。

![](https://upload-images.jianshu.io/upload_images/4236553-87cbd02b8f580397.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 上述操作没有考虑到另一种情况：leader 不知道自己已经被罢黜了。如果 leader 被罢黜，返回的就是脏数据。

如下图：

![](https://upload-images.jianshu.io/upload_images/4236553-ac73b3ef634e8d7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如何解决？
1. 领导人必须有关于被提交日志的最新信息。但任期开始时是个例外，他可能不知道那哪些是已经被提交的。所以在任期开始时需要提交一条空白的日志条目来实现。
2. 领导人在处理只读请求前，必须检查自己是否被废黜了（不能有脏数据），所以要先和集群的大多数节点交换一次心跳来处理这个问题。同时，领导人也可以通过依赖心跳机制来实现租约，但是这种方法依赖时间的正确性。

## Summary 

客户端怎么做：

客户端需要提供每条消息的唯一序列号，同时，如果是第一次访问，需要处理服务器可能返回的 leader  address 进行重试。

如果 leader 宕机，也需要进行重试。

***

服务端怎么做：

在写请求时，非 leader 不能处理写请求，需要返回当前 leader 信息给客户端。
在写请求时，leader 需要校验消息的序列号。
在读请求时，leader  需要通过心跳验证自己是否是 leader。同时，leader 需要有关于日志被提交时的最新信息，这通常能够保证，除了在任期开始时。所以， Raft 通过让 leader 在任期开始时提交一个空白的日志条目实现。

## 参考 
[英文 paper  pdf 地址](https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf)

[Raft paper 中文翻译 —— 寻找一种易于理解的一致性算法（扩展版）](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)

[Raft 作者讲解视频](https://www.youtube.com/watch?v=YbZ3zDzDnrw&feature=youtu.be)

[Raft 作者讲解视频对应的 PPT](http://www2.cs.uh.edu/~paris/6360/PowerPoint/Raft.ppt)

[一个简单的讲解 Raft 协议的动画](http://thesecretlivesofdata.com/raft/)
