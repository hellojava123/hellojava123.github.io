---
layout: post
title: Mac 搭建 Redis Sentinel 集群
date: 2018-08-09 00:01:01.000000000 +09:00
---
## 1. 创建目录并启动主从节点

#### 1. 主节点

执行以下命令：

```shell
➜  ~ mkdir redisSentinel
➜  ~ cd redisSentinel 
➜  redisSentinel mkdir master
➜  redisSentinel cp /usr/local/redis-3.2.8/redis.conf master/redis.conf
```

此时主节点目录已经创建好了，然后，配置主节点配置：

```yml
port 8000
daemonize yes
```

就简单修改这两个配置好了。
然后自动这个主节点：

```shell
➜  redisSentinel cd master 
➜  master redis-server redis.conf 
```

检查一下：

```shell
➜  master redis-cli -p 8000 ping
PONG
```

主节点就好了，接下来创建从节点。

****
#### 2. 从节点

```shell
➜  master cd ..
➜  redisSentinel mkdir slave
```

拷贝配置文件到从节点中：

```shell
➜  redisSentinel cp master/redis.conf slave/redis-8001.conf
➜  redisSentinel cp master/redis.conf slave/redis-8002.conf
```

这里我们将两个从节点的端口定为 8001 和 8002，然后修改配置。

```yml
port 800X
damonnize yes
slaveof 127.0.0.1 8000
```

加入了一个 slaveof 配置，表明该节点的主节点是 127.0.0.1:8000.

然后启动：

```shell
➜  redisSentinel cd slave 
➜  slave redis-server redis-8001.conf 
➜  slave redis-server redis-8002.conf
```

现在主从节点已经关联好了，现在通过 info 命令看看是否成功。

```shell
➜  slave redis-cli -p 8000 info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=8001,state=online,offset=505,lag=1
slave1:ip=127.0.0.1,port=8002,state=online,offset=505,lag=1
master_repl_offset:505
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:504
```


进入了 8000 主节点中查看，看见 role 是 master，并且有 2 个从节点，8001 和 8002. 成功！

接着开始部署哨兵节点.


## 2. 部署哨兵节点

首先创建配置文件：

```shell
➜  slave cd ..
➜  redisSentinel mkdir sentinel
➜  redisSentinel cd sentinel 
➜  sentinel cp /usr/local/redis-3.2.8/redis.conf redis-sentinel-26379.conf
➜  sentinel cp /usr/local/redis-3.2.8/redis.conf redis-sentinel-26380.conf
➜  sentinel cp /usr/local/redis-3.2.8/redis.conf redis-sentinel-26381.conf
```

哨兵节点的默认端口是 26739。我们这里弄了 3 个哨兵节点。

然后修改配置文件：

```shell
port 26379
daemonize yes
sentinel monitor mymaster 127.0.0.1 8000 2
```

这里比较关键的是：`sentinel monitor mymaster 127.0.0.1 8000 2`,这个配置表示该哨兵节点需要监控 8000 这个主节点， 2 代表着判断主节点失败至少需要 2 个 Sentinel 节点同意。

启动 Sentinel 节点：

```shell
➜  sentinel redis-sentinel redis-sentinel-26379.conf 
➜  sentinel redis-sentinel redis-sentinel-26380.conf
➜  sentinel redis-sentinel redis-sentinel-26381.conf
```

启动成功后，看看哨兵的相关信息：

```shell
➜  sentinel redis-cli -p 26380 info sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=127.0.0.1:8000,slaves=2,sentinels=3
```

通过进入 26380 节点，使用 info Sentinel 命令，看到有个 master 节点，名称是 mymaster，地址是我们刚刚配置的 8000， 从节点有 2 个，哨兵有 3 个。

此时一个高可用的 Redis 集群就搭建好了。

当然如果是生产环境，所有实例建议部署在不同的机器上。

部署后的拓扑图如下：

![](https://upload-images.jianshu.io/upload_images/4236553-77806db5e7db52bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 3. 使用 Jedis 测试

先在命令行往主节点写入一条数据:

```shell
➜  cachecloud git:(master) ✗ redis-cli -p 8000 set "hello" "world"
OK
```

Jedis 代码测试：

```java
  public static void main(String[] args) {

    Set<String> sentinelSet = new HashSet<>();
    sentinelSet.add("127.0.0.1:26379");
    sentinelSet.add("127.0.0.1:26380");
    sentinelSet.add("127.0.0.1:26381");

    String masterName = "mymaster";

    JedisSentinelPool sentinelPool = new JedisSentinelPool(masterName, sentinelSet, new GenericObjectPoolConfig(),
        10000, 10000, null, Protocol.DEFAULT_DATABASE);
    System.out.println(sentinelPool.getResource().get("hello"));
  }
```
结果：

```java
world
```

成功！！！


## 4. 总结

 Sentinel  节点实际上就是个特殊的 Redis 节点。
回头看看搭建过程：
1. 搭建主从节点，通过修改配置文件，关联主从节点。
2. 搭建哨兵节点集群，修改配置文件，通过 slaveof 监控主节点。

就好啦。























































































