---
layout: post
title: 异步 Servlet 和同步 Servlet 的性能测试
date: 2018-07-10 12:11:11.000000000 +09:00
---
## 前言

最近在看 Servlet 3 的异步特性，在网上看了一些文章，有些不解，遂做了一些测试，测试的模拟场景：**Web 容器工作线程较少，接口逻辑复杂耗时**。

这个场景是大部分的业务场景，例如：Tomcat 的工作线程是 200，业务接口需要远程 RPC 调用，或者访问数据库进行复杂计算等耗时操作。

所以，还是比较有参考价值的。



## 如何测试？

工具：idea，jmeter。

**服务端 idea**：SpringBoot 默认容器 Tomcat，修改线程数为 2，没错就是 2，为了放大工作线程较少的 场景。同时，准备两个接口：异步接口和同步接口，分别模拟耗时 500ms。

代码如下：

1, 异步接口
 
```java 
@ResponseBody
@Controller
@RequestMapping("async")
public class AsyncTestDemo {

  static ExecutorService workerPool = Executors.newFixedThreadPool(1000);

  @RequestMapping("async")
  public DeferredResult<String> async() {
    DeferredResult<String> defer = new DeferredResult<>((long) 120000);
    workerPool.submit(() -> {
      try {
        Thread.sleep(500);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      defer.setResult("hello async");
    });
    return defer;
  }
}
```

2, 同步接口


```java
@ResponseBody
@Controller
@RequestMapping("sync")
class SyncTestDemo{

  static ExecutorService workerPool =  Executors.newFixedThreadPool(1000);

  @RequestMapping("/sync")
  public String sync() throws ExecutionException, InterruptedException {
    Future<String> future = workerPool.submit(() ->
    {
      Thread.sleep(500);
      return "hello sync";
    });
    return future.get();
  }
}
```


3, Tomcat 配置：


```properties
server.port=8081
server.tomcat.max-threads=2
```

****
**客户端 jmeter：开启 200 线程分别访问两个接口。得到聚合报告**。

![线程组设置：200 线程](https://upload-images.jianshu.io/upload_images/4236553-83078c3561d7f1fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![异步请求设置](https://upload-images.jianshu.io/upload_images/4236553-552f0a8aa1b8993e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![同步请求设置](https://upload-images.jianshu.io/upload_images/4236553-74d935e1f7dd7adf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![聚合报告](https://upload-images.jianshu.io/upload_images/4236553-7f072d720b200574.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从聚合报告可以看出，两者差距巨大。
平均响应时间相差 45 倍，90% 的响应时间相差 73 倍，99% 的响应时间相差 66 倍。

同步 Servlet 只有在最初的 2 个请求中，响应时间在 500ms 左右，之后，由于线程数不够，后面的请求开始线性延时，最大响应时间为 44 秒（最后一个请求）。同时，错误率达到 11.50%。平均响应时间为 24 秒。

而异步 Servlet 则非常的稳定，错误率为 0 ，所有的请求响应时间都控制在 500ms 到 680ms 之间，平均时间为 541 ms，由于 Tomcat 线程数只有 2 个，这个结果可以接受。90% 的响应时间也是在 600 毫秒左右，属于正常水平。

## Summary 

异步 Servlet 发布已久，但似乎 Java 社区使用的人还是不多，不知为何？但异步带来的性能提升不言而喻。在服务器线程数较少，业务耗时的场景下，异步能明显提高系统吞吐量，线程数之外的请求不会像同步请求一样被拖慢。

实际上，这和 Netty 的最佳实践是类似的，永远不要在 IO 线程上做耗时任务，原则是：耗时任务丢进业务线程池，异步操作结束从 IO 线程返回或者直接返回。

防止 IO 线程阻塞，影响后面的请求。




