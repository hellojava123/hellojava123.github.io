---
layout: post
title: Jedis 如何支持 Sentinel
date: 2018-08-05 00:01:01.000000000 +09:00
---
## 前言

Jedis 作为 Java 世界 Redis 的老牌客户端，很好的支持了 Sentinel，例如 Sentinel 的故障转移功能。

## 代码

Jedis 提供了一个 Sentinel 构造方法：

```java
  public JedisSentinelPool(String masterName, Set<String> sentinels,
      final GenericObjectPoolConfig poolConfig, final int connectionTimeout, final int soTimeout,
      final String password, final int database, final String clientName) {
    this.poolConfig = poolConfig;
    this.connectionTimeout = connectionTimeout;
    this.soTimeout = soTimeout;
    this.password = password;
    this.database = database;
    this.clientName = clientName;

    HostAndPort master = initSentinels(sentinels, masterName);
    initPool(master);
  }
```

该构造方法做了 2件事情，初始化 Sentinel 和 Pool。我们先看看 initSentinel。

```java
 private HostAndPort initSentinels(Set<String> sentinels, final String masterName) {
    HostAndPort master = null; 
    boolean sentinelAvailable = false;

    for (String sentinel : sentinels) {// 遍历 Sentinel 节点
      final HostAndPort hap = HostAndPort.parseString(sentinel);// 解析 String 为 ip port

      Jedis jedis = null;
      try {
        jedis = new Jedis(hap.getHost(), hap.getPort());// 创建 Jedis 对象 
        // 执行 get-master-addr-by-name masterName 获取主节点信息
        List<String> masterAddr = jedis.sentinelGetMasterAddrByName(masterName);
        // 标识符：Sentinel 存在
        sentinelAvailable = true;
        // 如果主节点是空，或者返回长度不等于 2，跳过此 Sentinel
        if (masterAddr == null || masterAddr.size() != 2) {
          continue;
        }
        // 如果成功获取主节点，解析字符串成 HostAndPort 对象
        master = toHostAndPort(masterAddr);
        break;
      } catch (JedisException e) {
      } finally {
        if (jedis != null) {
          jedis.close();// 每次循环归还 Redis 连接
        }
      }
    }
    // 如果 master 是 null，抛出异常
    if (master == null) {
      if (sentinelAvailable) {// 细化异常，这个是 Sentinel 有问题
        throw new JedisException("Can connect to sentinel, but " + masterName
            + " seems to be not monitored...");
      } else {
        throw new JedisConnectionException("All sentinels down, cannot determine where is "
            + masterName + " master is running...");
      }
    }
    // 遍历 哨兵
    for (String sentinel : sentinels) {
      final HostAndPort hap = HostAndPort.parseString(sentinel);
      // 创建 master 监听器线程
      MasterListener masterListener = new MasterListener(masterName, hap.getHost(), hap.getPort());
      masterListener.setDaemon(true);// 后台线程
      masterListeners.add(masterListener);// 添加到监听集合，后期优雅关闭
      masterListener.start();// 启动线程
    }

    return master;
  }
```

方法很长，简单说说逻辑：
1. 遍历 Sentinel 字符串
2. 根据字符串生成 HostAndPort 对象，然后创建一个 Jedis 对象。
3. 使用 Jedis 对象发送 `get-master-addr-by-name masterName` 命令，得到 master 信息。
4. 得到 master 信息后，再次遍历哨兵集合，为每个哨兵创建一个线程，监听哨兵的发布订阅消息，消息主题是  `+switch-master`. 当主节点发生变化时，将通过 pub/sub 通知该线程，该线程将更新 Redis 连接池。

看看这个线程的主要内容：

```java
@Override
public void run() {
  // flag
  running.set(true);
  // 死循环
  while (running.get()) {
    //创建一个 Jedis对象
    j = new Jedis(host, port);
    try {
      // 继续检查
      if (!running.get()) {
        break;
      }
      // jedis 对象，通过 Redis pub/sub 订阅 switch-master 主题
      j.subscribe(new JedisPubSub() {
        @Override
        public void onMessage(String channel, String message) {
          // 分割字符串  
          String[] switchMasterMsg = message.split(" ");
          // 如果长度大于三
          if (switchMasterMsg.length > 3) {
            // 且 第一个 字符串的名称和当前 masterName 发生了 switch
            if (masterName.equals(switchMasterMsg[0])) {
              // 重新初始化连接池（第 4 个和 第 5 个）
              initPool(toHostAndPort(Arrays.asList(switchMasterMsg[3], switchMasterMsg[4])));
            } else {
            }
          } else {
          }
        }
      }, "+switch-master");

    } catch (JedisConnectionException e) {
      // 如果连接异常
      if (running.get()) {
        try {
          // 默认休息 5 秒
          Thread.sleep(subscribeRetryWaitTimeMillis);
        } catch (InterruptedException e1) {
        }
      } else {
      }
    } finally {
      j.close();
    }
  }
}
```

该方法已经写了很多注释，稍微说下逻辑：根据哨兵的 host 和 port 创建一个 jedis 对象，然后，这个 jedis 对象订阅了 pub/sub 消息，，消息的主题是 "+switch-master" ，如果收到消息了，就执行 onMessage 方法，该方法会根据新的  master 信息重新初始化 Redis 连接池。

那么如何初始化连接池的呢？

```java
private void initPool(HostAndPort master) {
  // 比较 host  + port，如果不相等，就重新初始化
  if (!master.equals(currentHostMaster)) {
    // 修改当前 master
    currentHostMaster = master;
    if (factory == null) {
      factory = new JedisFactory(master.getHost(), master.getPort(), connectionTimeout,
          soTimeout, password, database, clientName, false, null, null, null);
      initPool(poolConfig, factory);
    } else {
      // 修改连接参数, 下次获取连接的时候，就可以生成新的连接
      factory.setHostAndPort(currentHostMaster);
      // 清空旧的连接池
      internalPool.clear();
    }
  }
}
```

事实上，在 Sentinel 构造器里面，也会调用这个方法，第一次调用的时候， factory 肯定是 null，第二次调用的时候，会设置 factory 的 hostAndPort 为新的 master 地址，然后清空原来的连接池。那么新的 getResource 方法就会从这个新的地址获取到新的连接了。

具体关于 JedisSentinelPool 的  getResource 方法就不细说了，大家可以自己看看，还是很简单的，

## 总结

可以看到 Sentinel 的通知客户端机制，是需要客户端进行配合的，客户端需要通过 Sentinel 的 pub/sub 机制订阅哨兵节点的 `+switch-master` 主题，当 master 改变的时候，会通过 pub 通知客户端，客户端此时就可以优雅的更新连接池。





















