---
layout: post
title: 深入理解-Tomcat-(二)-从宏观上理解-Tomcat-组件及架构
date: 2017-11-18 11:11:11.000000000 +09:00
---
![小姐姐镇楼](http://upload-images.jianshu.io/upload_images/4236553-b990907440a7b7ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这是我们自编译源码以来第一次总结 tomcat, 虽然不知从何说起, 但这笔不能停下来, 看了很多的文章和源码, 脑子里从最初的混混沌沌到现在的稍有头绪, 楼主想说, 不容易.

tomcat 异常复杂, 组件巨多. 我认为, 作为初学者, 我们不能直接进入源码, 过多的实现细节会掩盖抽象概念, 容易一叶障目不见泰山. 我个人认为<<How Tomcat Works>>这本书写的很好, 而我也想和这本书的角度一样, 尽力(注意:是尽力)去描述我所理解的 tomcat. 

而我学习tomcat 的路线是:
1. 宏观上理解 tomcat 组件及架构设计.
2. 从一个简单的例子来理解tomcat的底层设计, 到底是怎么做的.
3. 从 tomcat 的各个组件各个击破, 理解每个组件的设计.
4. 从2条路线去 debug, 验证之前的理论是否正确, 并深刻理解代码. 第一条路线是启动过程. 第二条路线是一个 http 请求到 Servlet 的 service() 方法的过程.
5. 总结 tomcat 的设计.

大概是这个样子, 所以按照我们的研究路线, 大概会有5到6篇文章来详细讲述.

我们的文章会配上源码, 所以, 如果同学们没有下载源码, 可以去楼主关于 tomcat 的第一篇文章去下载源码, 最好将2份源码都下载, 因为我们会将2分源码混合的分析.

so, 这是第一篇(虽然标题是二). 

##### 这篇文章我们主要讲什么?
###### 1. 什么是 tomcat?
      1. tomcat 的历史
      2. 和 tomcat 高度相关的 Servlet 是什么?是如何工作的?
###### 2. tomcat 是如何设计的? 有哪些所谓的组件?
      1.  tomcat 源码目录的解释
      2.  tomcat 整体框架的层次结构和架构图
      3. 分析每个组件, 包括 Connector, Container, Component
      4. 从接口和类的角度(UML 类图)看架构.
###### 3. 总结
***
## 什么是 tomcat ?
我们还是引用一下维基百科:
> Tomcat是由Apache软件基金会下属的Jakarta项目开发的一个Servlet容器，按照Sun Microsystems提供的技术规范，实现了对[Servlet](https://zh.wikipedia.org/wiki/Servlet)和[JavaServer Page](https://zh.wikipedia.org/wiki/JavaServer_Page)（[JSP](https://zh.wikipedia.org/wiki/JSP)）的支持，并提供了作为Web服务器的一些特有功能，如Tomcat管理和控制平台、安全域管理和Tomcat阀等。由于Tomcat本身也内含了一个[HTTP](https://zh.wikipedia.org/wiki/HTTP)[服务器](https://zh.wikipedia.org/wiki/%E6%9C%8D%E5%8A%A1%E5%99%A8)，它也可以被视作一个单独的[Web服务器](https://zh.wikipedia.org/wiki/Web%E6%9C%8D%E5%8A%A1%E5%99%A8)。但是，不能将Tomcat和[Apache HTTP服务器](https://zh.wikipedia.org/wiki/Apache_HTTP%E6%9C%8D%E5%8A%A1%E5%99%A8)混淆，[Apache HTTP服务器](https://zh.wikipedia.org/wiki/Apache_HTTP%E6%9C%8D%E5%8A%A1%E5%99%A8)是一个用C语言实现的HTTP[Web服务器](https://zh.wikipedia.org/wiki/Web%E6%9C%8D%E5%8A%A1%E5%99%A8)；这两个HTTP web server不是捆绑在一起的。Apache Tomcat包含了一个配置管理工具，也可以通过编辑XML格式的配置文件来进行配置。


##### 简而言之: tomcat 是一个接受 http 请求并解析 http 请求并反馈客户端的一个应用程序.

tomcat 可以说是 sun Servlet 的一个官方参考实现, 因为 tomcat 刚开始就是 sun 开发的,后来捐献给了 apache 基金会.


在我们理解 tomcat 之前, 我们得了解一下tomcat 所实现的 Servlet 是什么东东? 

那要从互联网大潮兴起开始讲起, 我们知道, 网页不止有静态的还有动态的, 而在刚开始的时候, 常见的实现动态网页的技术就是 CGI, 但是作为 Java 的发明人, SUN|肯定要搞一个超级大风头, 让整个世界一下子认识了 Java, 不过很快悲催的发现 Applet 其实用途不大, 眼看着互联网开始流行, 一定要搭上千载难逢的快车啊.

于是, Servlet 就诞生了, Servlet 其实就是 SUN 为了让 Java 能实现动态的可交互的网页, 从而进入 web 编程的领域而定义的一套标准. 

这套标准是这么说的: 你想用 Java 开发动态网页, 可以定义一个自己的" Servlet", 但一定要实现我的 HTTPServlet 接口, 然后重载 doGet(), doPost()方法. 用户从流浪器 GET 的时候, 调用 doGet 方法, 从流浪器向服务器发送表单数据的时候, 调用 doPost 方法, 如果你想访问用户从浏览器传递过来的参数, 用 HttpServletRequest 对象就好了, 里面有 getParameter, getQueryString 方法, 如果你处理完了, 想向浏览器返回数据, 用 HttpServletResponse 对象调用 getPrintWriter 方法就可以输出数据了. 

如果你想实现一个购物车, 需要 session, 很简单, 从 HttpServletRequest 调用 getSession 方法就可以了.

### 那么..... Servlet 是如何工作的
servlet 是一个复杂的系统, 但是他有三个基本任务, 对每个请求, servlet 容器会为其完成以下3个操作:
 1. 创建一个 Request 对象, 用可能会在调用的 Servlet 中使用到的信息填充该 Request 对象, 如参数, 头, cookie, 查询字符串, URI 等, Request 对象是 javax.servlet.ServletRequest 接口或 javax.servlet.ServletRequest 接口的一个实例.
 2. 创建一个调用 Servlet 的 Response 对象, 用来向 Web 客户端发送响应. response 对象是 javax.servlet.http.ServletResponse 接口或 javax.servlet.ServletResponse 接口的一个实例;
 3. 调用 Servlet 的 service 方法, 将 request 对象和 response 对象作为参数传入, Servlet 从 request 对象中读取信息, 并通过 response 对象发送响应信息.

你写了一个" Servlet", 接下来要运行, 你就发现没法通过 java 直接运行了, 你需要一个能够运行 Servlet 的容器, 这个容器 Sun 最早实现了一个, 叫 Java Web Server, 1999年捐给了 Apache Software foundation, 就改名叫 Tomcat. 这就是 Tomcat 的由来.

所以, Tomcat 就是一个 Servlet 容器, 能接收用户从浏览器发来的请求, 然后转发给 Servlet 处理, 把处理完的响应数据发回浏览器.

但是 Servlet 输出 Html, 还是采用了老的 CGI 方式 , 是一句一句输出, 所以,编写和修改 html 非常不方便, 于是, Java Server Pages(JSP) 就来救急了, JSP 并没有增加任何本质上不能用 Servlet 实现的功能, 实际上 JSP 在运行之前, 需要先编译成 Servlet, 然后才执行的. 

但是, 在 JSP 中编写静态 HTML 更加方便, 不必再用 println 语句来输出每一行HTML 代码, 更重要的是, 借助内容和外观的分离,页面制作中不同性质的任务可以方便的分开: 比如, 有页面设计者进行 HTML 设计, 同时流出供 Java 程序插入动态内容的空间. 

Tomcat 能运行 Servlet, 当然能运行 JSP 了.

既然是 Web 服务器, tomcat 除了能运行 Servlet 和 JSP 之外, 也能像 Apache/nginx 一样, 支持静态 html, 图片, 文档的访问, 只是性能差一点,在实际的应用中, 一般是 Apache/nginx 作为负载均衡服务器和静态资源服务器放在最前端, 后面是 tomcat 组成的集群.

如果用户请求的是静态资源, nginx 直接搞定, 如果是动态资源(如xxx.jsp), nginx 就会按照一定的算法转发的某个 tomcat 上, 达到负载均衡的目的.

***

##   Tomcat 是如何设计的? 有哪些所谓的组件?

tomcat 的主体是 Catalina, catalina 是一个成熟的软件, 设计和开发的十分优雅, 功能结构也是模块化的. 我们之前说 `Servlet 是如何工作的?`中提到了 Servlet 容器的任务, 基于这些任务可以将 Catalina 划分为两个模块: **连接器(connector)和容器(container)**.

**注意**: 连接器和 Servlet 容器是一对多的关系.

连接器负责将一个请求与容器相关联, 他的工作为它接收到的每个 http 请求创建一个 Request 对象和一个 Response 对象, 然后, 他将处理过程交给容器.  容器从连接器中接收到 Request 对象和 Response 对象, 并负责调用相应的 service 方法.

但是请记住, 上面所描述的处理过程只是 Catalina 容器处理请求的的整个过程的一小部分, 犹如冰山的一角, 在容器中还包括很多其他的事情要做, 例如, 在容器调用相应的 Servlet 的 service 方法之前, 他必须先载入该 Servlet 类, 对用户进行身份验证(如果有必要的话), 为用户更新会话信息等, 因此, 当你发现容器使用了很多不同的模块来处理这些事情时, 请不要它惊讶, 例如, **管理器模块**用来处理用户会话信息, **类载入器模块**用来载入所需的 Servlet 类.

###### 那么我们来看看 Tomcat 源码的目录结构和每个目录是起什么作用的:
![dir](http://upload-images.jianshu.io/upload_images/4236553-a657a0b26724dad2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
*****
#### Tomcat 整体的框架层次：4个层次, 其中 Connector 和 Container 是最重要的.
1. Server 和 Service
2. Connector
   * HTTP
   * AJP (apache 私有协议，用于tomcat和apache静态服务器通信)
3. Container
   * Engine
   * Host
   * Context
   * Wrapper
4. Component
   * Manager （管理器）
   * logger （日志管理）
   * loader （载入器）
   * pipeline (管道)
   * valve （管道中的阀）

##### 下面我们来看一张有名的关于 tomcat 的结构图:
![tomcat 结构图](http://upload-images.jianshu.io/upload_images/4236553-29ffff1d2e70ceee.JPG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**注意**: *Server*是整个Tomcat组件的容器，包含一个或多个Service。 *Service*：Service是包含Connector和Container的集合，Service用适当的Connector接收用户的请求，再发给相应的Container来处理。

> 从这张图中可以看到，Tomcat的核心组件就两个Connector和Container（后面还有详细说明），多个Connector+一个Container构成一个Service，Service就是对外提供服务的组件，有了Service组件Tomcat就可以对外提供服务了，但是光有服务还不行，还得有环境让你提供服务才行，所以最外层的Server就为Service提供了生存的土壤。那么这些个组件到底是干嘛用的呢？Connector是一个连接器，主要负责接收请求并把请求交给Container，Container就是一个容器，主要装的是具体处理请求的组件。Service主要是为了关联Container与Connector，一个单独的Container或者一个单独的Connector都不能完整处理一个请求，只有两个结合在一起才能完成一个请求的处理。Server这是负责管理Service集合，从图中我们看到一个Tomcat可以提供多种服务，那么这些Serice就是由Server来管理的，具体的工作包括：对外提供一个接口访问Service，对内维护Service集合，维护Service集合又包括管理Service的生命周期、寻找一个请求的Service、结束一个Service等
*****
#### Connector 组件:
> Tomcat都是在容器里面处理问题的， 而容器又到哪里去取得输入信息呢？ Connector就是专干这个的。 他会把从socket传递过来的数据， 封装成Request, 传递给容器来处理。 通常我们会用到两种Connector,一种叫http connectoer， 用来传递http需求的。 另一种叫AJP， 在我们整合apache与tomcat工作的时候，apache与tomcat之间就是通过这个协议来互动的。 （说到apache与tomcat的整合工作， 通常我们的目的是为了让apache 获取静态资源， 而让tomcat来解析动态的jsp或者servlet。）

总的来说，Connector就是解析Http或Ajp请求的。

*****
#### Container 组件:
我们刚刚看到, 容器从大到小分别是 Engine, Host, Context, Wrapper, 从左到右每个容器都是一对多关系, 也就是说, Engine 容器可以有多个 Host 容器, Host 容器可以有多个 Context 容器. Context 容器可以有多个 Wrapper 容器.
我们来看看每个组件的解释:
1. Container：可以理解为处理某类型请求的容器，处理的方式一般为把处理请求的处理器包装为Valve(阀门)对象，并按一定顺序放入类型为Pipeline(管道)的管道里。Container有多种子类型：Engine、Host、Context和Wrapper，这几种子类型Container依次包含，处理不同粒度的请求。另外Container里包含一些基础服务，如Loader、Manager和Realm。
2. Engine：Engine包含Host和Context，接到请求后仍给相应的Host在相应的Context里处理。
3. Host：就是我们所理解的虚拟主机。
4. Context：就是我们所部属的具体Web应用的上下文，每个请求都在是相应的上下文里处理的。
5. Wrapper：Wrapper是针对每个Servlet的Container，每个Servlet都有相应的Wrapper来管理。 可以看出Server、Service、Connector、Container、Engine、Host、Context和Wrapper这些核心组件的作用范围是逐层递减，并逐层包含。 下面就是些被Container所用的基础组件：
   * Loader：是被Container用来载入各种所需的Class。
   * Manager：是被Container用来管理Session池。
   * Realm：是用来处理安全里授权与认证。
***
####  Component 组件:
需求被传递到了容器里面， 在合适的时候， 会传递给下一个容器处理。而容器里面又盛装着各种各样的组件， 我们可以理解为提供各种各样的增值服务。比如:

1. manager: 当一个容器里面装了manager组件后，这个容器就支持session管理了， 事实上在tomcat里面的session管理, 就是靠的在context里面装的manager component. 

2. logger: 当一个容器里面装了logger组件后， 这个容器里所发生的事情， 就被该组件记录下来, 我们通常会在logs/ 这个目录下看见catalina_log.time.txt 以及localhost.time.txt和localhost_examples_log.time.txt。 这就是因为我们分别为：engin, host以及context(examples)这三个容器安装了logger组件， 这也是默认安装， 又叫做标配 .

3. loader: loader这个组件通常只会给我们的context容器使用，loader是用来启动context以及管理这个context的classloader用的。

4. pipline: pipeline是这样一个东西，使用的责任链模式.  当一个容器决定了要把从上级传递过来的需求交给子容器的时候， 他就把这个需求放进容器的管道(pipeline)里面去。 而需求傻呼呼得在管道里面流动的时候， 就会被管道里面的各个阀门拦截下来。 比如管道里面放了两个阀门。 第一个阀门叫做“access_allow_vavle”， 也就是说需求流过来的时候，它会看这个需求是哪个IP过来的， 如果这个IP已经在黑名单里面了，sure, 杀！ 第二个阀门叫做“defaul_access_valve”它会做例行的检查， 如果通过的话，OK， 把需求传递给当前容器的子容器。 就是通过这种方式， 需求就在各个容器里面传递，流动， 最后抵达目的地的了。

5. valve: 就是上面所说的阀门。

Tomcat里面大概就是这么些东西， 我们可以简单地这么理解tomcat的框架，它是一种自上而下， 容器里又包含子容器的这样一种结构。


## 换个角度再来看看 Tomcat 的总体架构:
* 面向组件架构
* 基于JMX
* 事件侦听

1. **面向组件架构** tomcat代码看似很庞大，但从结构上看却很清晰和简单，它主要由一堆组件组成，如Server、Service、Connector等，并基于JMX管理这些组件，另外实现以上接口的组件也实现了代表生存期的接口Lifecycle，使其组件履行固定的生存期，在其整个生存期的过程中通过事件侦听LifecycleEvent实现扩展。Tomcat的核心类图如下所示:
![核心类图](http://upload-images.jianshu.io/upload_images/4236553-648b485b5b7bdf06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面我们已经对每个类的功能进行了一些描述.包括 Catalina 类和 Server 类的作用, Service 类是如何关联 Connector 和 Container 的. 还有各个容器是上面关系. 包括容器中数据是如何从从管道( pipeline) 中传递, 还有一些组件, 比如类载入器, 管理器, 安全主体( realm);

2. 基于JMX Tomcat会为每个组件进行注册过程，通过Registry管理起来，而Registry是基于JMX来实现的，因此在看组件的init和start过程实际上就是初始化MBean和触发MBean的start方法，会大量看到形如： Registry.getRegistry(null, null).invoke(mbeans, "init", false); Registry.getRegistry(null, null).invoke(mbeans, "start", false); 这样的代码，这实际上就是通过JMX管理各种组件的行为和生命期。

那么, 什么是 JMX 呢?
> JMX 即 Java Management Extensions(JMX 规范), 是用来对 tomcat 进行管理的. tomcat 中的实现是 commons modeler 库, Catalina 使用这个库来见哈编写托管 Bean 的工作. 托管 Bean 就是用来管理 Catalina 中其他对象的 Bean. 

3. 事件侦听(观察者模式/事件驱动) 各个组件在其生命期中会有各种各样行为，而这些行为都有触发相应的事件，Tomcat就是通过侦听这些时间达到对这些行为进行扩展的目的。在看组件的init和start过程中会看到大量如： lifecycle.fireLifecycleEvent(AFTER_START_EVENT, null);这样的代码，这就是对某一类型事件的触发，如果你想在其中加入自己的行为，就只用注册相应类型的事件即可。


****
### 最后:
 我们已经从宏观上详细分析了 Tomcat 的架构和组件等一些抽象的概念, 可能大家还比较懵逼, 因为我们完全是纸上谈兵, 没有真刀真枪实战源码, 所以, 接下来, 让我们带着我们准备好的源码一起撕开 Tomcat 的衣服, 看看她到底是什么样子的.   

还是像刚开始说的: 
1. 宏观上理解 tomcat 组件及架构设计.
2. 从一个简单的例子来理解tomcat的底层设计, 到底是怎么做的.
3. 从 tomcat 的各个组件各个击破, 理解每个组件的设计.
4. 从2条路线去 debug, 验证之前的理论是否正确, 并深刻理解代码. 第一条路线是启动过程. 第二条路线是一个 http 请求到 Servlet 的 service() 方法的过程.
5. 总结 tomcat 的设计.


现在我们已经完成了第一步, 宏观上理解了有哪些组件和他的架构设计. 后面的学习我们按照计划来执行, 不要操之过急, 看源码最重要的就是沉的下心. 我们会
先从组件和接口分析, 比如 Connector 和 Container 接口, 还有4大容器接口和管道阀门, 还有 生命周期接口.他们分别使用了什么设计模式,还有tomcat 钩子的使用,  包括tomcat著名的类加载机制, 为什么违背双亲委任模型等等. 这些内容都将是什么重点关注的.


好了, 今天就到这里, good luck !!!!!









