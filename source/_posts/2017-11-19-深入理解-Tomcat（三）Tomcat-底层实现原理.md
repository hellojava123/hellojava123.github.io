---
layout: post
title: 深入理解-Tomcat（三）Tomcat-底层实现原理
date: 2017-11-19 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-593d12bb769a8353.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


又是一个周末，这篇文章将从一个简单的例子来理解tomcat的底层设计;


**本文将介绍 Java Web 服务器是如何运行的, Web 服务器也称为超文本传输协议( HyperText Transfer Protocol, HTTP)服务器, 因为它使用 Http 与其客户端(通常是 Web 浏览器)进行通信, 基于 Java 的 Web 服务器会使用两个重要的类: java.net.Socket 类和 java.net.ServerSocket 类, 并通过发送 Http 消息进行通信. 我们先花一些篇幅介绍 Http 协议（如果同学们熟悉HTTP协议可直接跳过）和这两个类, 然后写一个简单的 Web 服务器.**

#### HTTP

Http : Http 允许 Web 服务器和浏览器通过 Internet 发送并接受数据, 是一种基于"请求---响应"的协议, 客户端请求一个文件, 服务器端对该请求进行响应. Http 使用可靠的 tcp 连接, tcp 协议默认使用 tcp 80端口, http协议的第一个版本是 http/0.9, 后来被 http/1.0取代, 随后 http/1.0又被http/1.1取代, http/1.1 定义域 RFC(Request for Comment, 请求注解)2616中. 

如果各位对 Http1.1 有更多兴趣, 请阅读 RFC 2616.

在 Http 中, 总是由客户端通过建立连接并发送 http 请求来初始化一个事务的. Web 服务器端并不负责联系客户端或建立一个到客户端的回调连接.客户端或服务器端可提前关闭连接, 例如, 当使用 Web 浏览器浏览网页时, 可以单击浏览器上的 stop 按钮来停止下载文件, 这样就有效的关闭了一个 Web 服务器的 http 连接.


#### HTTP 请求
一个 HTTP 请求包含以下三部分:
* 请求方法----统一资源标识符(Uniform Resource Identifier, URI)------协议/版本
* 请求头
* 实体

下面是一个 HTTP 请求的例子： 

```
POST /examples/default.jsp HTTP/1.1 
Accept: text/plain; text/html 
Accept-Language: en-gb 
Connection: Keep-Alive 
Host: localhost 
User-Agent: Mozilla/4.0 (compatible; MSIE 4.01; Windows 98)
Content-Length: 33 Content-Type: application/x-www-form-urlencoded Accept-Encoding: gzip, deflate 

lastName=Franks&firstName=Michael  

```

  方法—统一资源标识符(URI)—协议/版本出现在请求的第一行。  
  `POST /examples/default.jsp HTTP/1.1    `  
  
  这里 POST 是请求方法，`/examples/default.jsp` 是 URI，而 HTTP/1.1 是协议/版本部分。     每个 HTTP 请求可以使用 HTTP 标准里边提到的多种方法之一。HTTP 1.1 支持 7 种类型的请 求：GET, POST, HEAD, OPTIONS, PUT, DELETE 和 TRACE。GET 和 POST 在互联网应用里边最普遍使用的。     
  
  URI 完全指明了一个互联网资源。URI 通常是相对服务器的根目录解释的。因此，始终一斜 线/开头。统一资源定位器(URL)其实是一种 URI(查看 http://www.ietf.org/rfc/rfc2396.txt)
来的。该协议版本代表了正在使用的 HTTP 协议的版本。 

 请求的头部包含了关于客户端环境和请求的主体内容的有用信息。例如它可能包括浏览器设 置的语言，主体内容的长度等等。每个头部通过一个回车换行符(CRLF)来分隔的。     
 
 对于 HTTP 请求格式来说，头部和主体内容之间有一个回车换行符(CRLF)是相当重要的。CRLF 告诉HTTP服务器主体内容是在什么地方开始的。在一些互联网编程书籍中，CRLF还被认为是HTTP 请求的第四部分。     
 
 在前面一个 HTTP 请求中，主体内容只不过是下面一行： 
 
`lastName=Franks&firstName=Michael    `

实体内容在一个典型的 HTTP 请求中可以很容易的变得更长。 


### HTTP 响应

类似于 HTTP 请求，一个 HTTP 响应也包括三个组成部分：  
* 方法—统一资源标识符(URI)—协议/版本 
* 响应的头部  
* 主体内容      
 
 下面是一个 HTTP 响应的例子：
```html
HTTP/1.1 200 OK 
Server: Microsoft-IIS/4.0 
Date: Mon, 5 Jan 2004 13:13:33 GMT 
Content-Type: text/html 
Last-Modified: Mon, 5 Jan 2004 13:13:12 GMT 
Content-Length: 112 
 
<html> 
    <head> 
        <title>HTTP Response Example</title> 
    </head> 
    <body> 
        Welcome to Brainy Software 
    </body> 
</html> 
```

响应头部的第一行类似于请求头部的第一行。第一行告诉你该协议使用 HTTP 1.1，请求成 功(200=成功)，表示一切都运行良好。 

响应头部和请求头部类似，也包括很多有用的信息。响应的主体内容是响应本身的 HTML 内 容。头部和主体内容通过 CRLF 分隔开来。 


### Socket 类  

 套接字是网络连接的一个端点。套接字使得一个应用可以从网络中读取和写入数据。放在两 个不同计算机上的两个应用可以通过连接发送和接受字节流。为了从你的应用发送一条信息到另 一个应用，你需要知道另一个应用的 IP 地址和套接字端口。在 Java 里边，套接字指的是`java.net.Socket`类。 
 
要创建一个套接字，你可以使用 Socket 类众多构造方法中的一个。其中一个接收主机名称 和端口号:
```java
public Socket (java.lang.String host, int port)
```

在这里主机是指远程机器名称或者 IP 地址，端口是指远程应用的端口号。例如，要连接 yahoo.com 的 80 端口，你需要构造以下的 Socket 对象:

```java
new Socket ("yahoo.com", 80);
```

一旦你成功创建了一个 Socket 类的实例，你可以使用它来发送和接受字节流。要发送字节 流，你首先必须调用Socket类的getOutputStream方法来获取一个java.io.OutputStream对象。 要 发 送 文 本 到 一 个 远 程 应 用 ， 你 经 常 要 从 返 回 的 OutputStream 对 象 中 构 造 一 个 java.io.PrintWriter 对象。要从连接的另一端接受字节流，你可以调用 Socket 类的 getInputStream 方法用来返回一个 java.io.InputStream 对象。 


### ServerSocket 类 


 Socket 类代表一个客户端套接字，即任何时候你想连接到一个远程服务器应用的时候你构 造的套接字，现在，假如你想实施一个服务器应用，例如一个 HTTP 服务器或者 FTP 服务器，你 需要一种不同的做法。这是因为你的服务器必须随时待命，因为它不知道一个客户端应用什么时 候会尝试去连接它。为了让你的应用能随时待命，你需要使用 java.net.ServerSocket 类。这是 服务器套接字的实现。
 
 
  ServerSocket 和 Socket 不同，服务器套接字的角色是等待来自客户端的连接请求。一旦服 务器套接字获得一个连接请求，它创建一个 Socket 实例来与客户端进行通信。 
  
  
   要创建一个服务器套接字，你需要使用 ServerSocket 类提供的四个构造方法中的一个。你 需要指定 IP 地址和服务器套接字将要进行监听的端口号。通常，IP 地址将会是 127.0.0.1，也 就是说，服务器套接字将会监听本地机器。服务器套接字正在监听的 IP 地址被称为是绑定地址。 服务器套接字的另一个重要的属性是 backlog，这是服务器套接字开始拒绝传入的请求之前，传 入的连接请求的最大队列长度。 
   
 其中一个 ServerSocket 类的构造方法如下所示: 
 
 ```java
 public ServerSocket(int port, int backLog, InetAddress bindingAddress); 
 ```


### 应用程序  

如果同学们下载过我们在第一篇文章提供的源码（How Tomcat Works）的话， 我们可以看一看我们的目录：

![](http://upload-images.jianshu.io/upload_images/4236553-eb2b886a4c98ce94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 我们的 web 服务器应用程序放在 cxs01.pyrmont(编译的时候因为错误改名字了，也就懒得改回来了) 包里边，由三个类组成： 
* HttpServer  
* Request  
* Response

 这个应用程序的入口点(静态 main 方法)可以在 HttpServer 类里边找到。main 方法创建了 一个 HttpServer 的实例并调用了它的 await 方法。await 方法，顾名思义就是在一个指定的端 口上等待 HTTP 请求,处理它们并发送响应返回客户端。它一直等待直至接收到 shutdown 命令。    
![](http://upload-images.jianshu.io/upload_images/4236553-ad482afb9756566e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


 应用程序不能做什么，除了发送静态资源，例如放在一个特定目录的 HTML 文件和图像文件。 它也在控制台上显示传入的 HTTP 请求的字节流。不过，它不给浏览器发送任何的头部例如日期 或者 cookies。 

##### 下面我们来看看我们今天的重点，这三个类，也就是tomcat的雏形代码
###### HttpServer 类

  HttpServer 类代表一个 web 服务器，也就是程序的入口，看代码：
```java
public class HttpServer {
  public static final String WEB_ROOT =
    System.getProperty("user.dir") + File.separator  + "webroot";

  // 关闭命令
  private static final String SHUTDOWN_COMMAND = "/SHUTDOWN";

  // 是否关闭
  private boolean shutdown = false;

  public static void main(String[] args) {
    HttpServer server = new HttpServer();
    server.await();
  }
```

main 方法中创建了一个HttpServer对象，并调用了该对象的await方法。看名字，该方法应该是等待http请求之类的东东。我们来看看方法内部：
```java
public void await() {
    ServerSocket serverSocket = null;
    int port = 8080;
    try {
      // 创建一个socket服务器
      serverSocket =  new ServerSocket(port, 1, InetAddress.getByName("127.0.0.1"));
    }
    catch (IOException e) {
      e.printStackTrace();
      System.exit(1);
    }

    // 循环等待http请求
    while (!shutdown) {
      Socket socket = null;
      InputStream input = null;
      OutputStream output = null;
      try {
        // 阻塞等待http请求
        socket = serverSocket.accept();
        input = socket.getInputStream();
        output = socket.getOutputStream();

        // 创建一个Request对象用于解析http请求内容
        Request request = new Request(input);
        request.parse();

        // 创建一个Response 对象，用于发送静态文本
        Response response = new Response(output);
        response.setRequest(request);
        response.sendStaticResource();

        // 关闭流
        socket.close();

        //检查URI中是否有关闭命令
        shutdown = request.getUri().equals(SHUTDOWN_COMMAND);
      }
      catch (Exception e) {
        e.printStackTrace();
        continue;
      }
    }
  }
```

我们看到，该方法创建了一个Socket服务器，并循环阻塞监听http请求，当有http请求到来时， 该方法便创建一个Request对象，构造参数是socket获取的输入流对象， 用于读取客户端请求的数据并解析。 然后再创建一个Response对象，构造参数是socket的输出流对象， 并含有一个Request对象的成员变量。Response对象用于将静态页面发送给浏览器或者是其他的客户端。最后， 该方法校验请求中是否含有关闭命令的字符串，如果有，就停止服务器的运行。

这就是一个简单的服务器， 当我第一次看到的时候，心想： 真TMD简单啊。原来没那么复杂嘛。我想同学们心里想的跟我也一样吧。so， 不论多么庞大的代码，底层原理都是很简单的，只要我们学好了基础，学习起来就会轻松很多。

废话不多说，我们继续看看Request 是如何解析Http请求的吧。

##### Request 类
类结构图如下：
![image](http://upload-images.jianshu.io/upload_images/4236553-1690572b88d907ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 Request 类代表一个 HTTP 请求。从负责与客户端通信的 Socket 中传递过来 InputStream 对象来构造这个类的一个实例。你调用 InputStream 对象其中一个 read 方法来获 取 HTTP 请求的原始数据。其中最主要的方法就是parse 和 parseUri ，他们用于逐个解析每个从客户端传递过来的字节，我们先看parse方法：
```java
  public void parse() {
    // Read a set of characters from the socket
    StringBuffer request = new StringBuffer(2048);
    int i;
    byte[] buffer = new byte[2048];
    try {
      // 读取流中内容
      i = input.read(buffer);
    }
    catch (IOException e) {
      e.printStackTrace();
      i = -1;
    }
    for (int j=0; j<i; j++) {
     // 将每个字节转换为字符
      request.append((char) buffer[j]);
    }
    // 打印字符串
    System.out.print(request.toString());
    // 根据转换出来的字符解析URI
    uri = parseUri(request.toString());
  }
```
我们也看到该方法是十分的简单， 创建一个StringBuffer 对象，然后从流中读取字节，然后循环将字节转成字符写入到Stringbuffer对象中。最后传入到parseUri方法中进行解析。 

我们再看看parseUri方法， 这个方法中，我们前面学习的关于HTTP的知识会起作用：
```java
  private String parseUri(String requestString) {
    int index1, index2;
    // 找到第一个空格
    index1 = requestString.indexOf(' ');
    if (index1 != -1) {
      // 找到第二个空格
      index2 = requestString.indexOf(' ', index1 + 1);
      if (index2 > index1)
        // 截取第一个空格到第二个空格之间的内容
        return requestString.substring(index1 + 1, index2);
    }
    return null;
  }

```
该方法从请求行里边获得 URI。parseUri 方法搜索请求里边的第一个和第二个空格并从中获取 URI。
为什么是第一个空格和第二个空格之间的内容呢？我们看看前面的Http请求的例子：
```html
POST /examples/default.jsp HTTP/1.1 
Accept: text/plain; text/html 
Accept-Language: en-gb 
Connection: Keep-Alive 
Host: localhost 
User-Agent: Mozilla/4.0 (compatible; MSIE 4.01; Windows 98)
Content-Length: 33 Content-Type: application/x-www-form-urlencoded Accept-Encoding: gzip, deflate 

lastName=Franks&firstName=Michael  
```

我们看第一行：
![](http://upload-images.jianshu.io/upload_images/4236553-6a391bc0c6581bb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

POST 和 HTTP/1.1之间的就是我们需要的URI。so， 我们只需要将中间那段字符串截取就OK了。

我们总结一下Request类，这个类其实就是解析HTTP 消息头内容的，先将流中数据转成字节，然后将转成字符，最后将字符解析，得到自己感兴趣的内容。奏是这么简单。好了，我们再看看Response类。看看他是怎么实现的。

#### Response类
我们先看看这个类的结构图：
![](http://upload-images.jianshu.io/upload_images/4236553-f5da79e8a3018c13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Response 代表了Http请求中的一个响应。我们关注其中的 sendStaticResource 方法，看名字，该方法应该是发送静态资源给客户端。我们看看代码：
```java
  public void sendStaticResource() throws IOException {
    byte[] bytes = new byte[BUFFER_SIZE];
    FileInputStream fis = null;
    try {
      File file = new File(HttpServer.WEB_ROOT, request.getUri());
      if (file.exists()) {
        // 文件存在则从输出流中输出
        fis = new FileInputStream(file);
        int ch = fis.read(bytes, 0, BUFFER_SIZE);
        while (ch!=-1) {
          output.write(bytes, 0, ch);
          ch = fis.read(bytes, 0, BUFFER_SIZE);
        }
      }
      else {
        // 没有文件返回404
        String errorMessage = "HTTP/1.1 404 File Not Found\r\n" +
          "Content-Type: text/html\r\n" +
          "Content-Length: 23\r\n" +
          "\r\n" +
          "<h1>File Not Found</h1>";
        output.write(errorMessage.getBytes());
      }
    }
    catch (Exception e) {
      // thrown if cannot instantiate a File object
      System.out.println(e.toString() );
    }
    finally {
      if (fis!=null)
        fis.close();
    }
  }
```

可以看到，该方法也非常的简单， sendStaticResource 方法是用来发送一个静态资源，例如一个 HTML 文件。它首先通过传递 上一级目录的路径和子路径给 File 累的构造方法来实例化 java.io.File 类。

然后它检查该文件是否存在。假如存在的话，通过传递 File 对象让 sendStaticResource 构造一个 java.io.FileInputStream 对象。然后，它调用 FileInputStream 的 read 方法并把字 节数组写入 OutputStream 对象。请注意，这种情况下，静态资源是作为原始数据发送给浏览器 的。

假如文件并不存在，sendStaticResource 方法发送一个错误信息到浏览器

运行程序，启动HttpServer mian方法，使用Edge浏览器在地址栏敲入：http://localhost:8080/index.html
返回：
![](http://upload-images.jianshu.io/upload_images/4236553-0d74bdf2b346030d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

表示文件存在， 再看看我们的后台控制台：
![](http://upload-images.jianshu.io/upload_images/4236553-282d24fa71c9d9e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如期打印了http请求头中的内容。并且下面还请求了一张图片。

### 总结

至此，我们已经知道了一个简单的Web服务器是如何工作的。破除了我们之前的疑惑，实际上tomcat底层就是这么实现的，可能关于阻塞IO和非阻塞NIO会有区别，但总体上还是这个思路，然后其余的组件都是针对优化性能，提高扩展性来设计新的架构。所以，我们明白了底层设计，再去学习他的设计，就不会那么迷茫。从而感到泄气。毕竟每个夜晚，我们孤独的学习，不想徒劳无功。

好了，本文结束！！！ good luck ！！！






















