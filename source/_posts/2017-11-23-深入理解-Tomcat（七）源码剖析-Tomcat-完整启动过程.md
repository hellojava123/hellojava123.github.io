---
layout: post
title: 深入理解-Tomcat（七）源码剖析-Tomcat-完整启动过程
date: 2017-11-23 11:11:11.000000000 +09:00
---
![](http://upload-images.jianshu.io/upload_images/4236553-b69d11040c663699.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 前言

这是我们分析 Tomcat 的第七篇文章，前面我们依据启动过程理解了类加载过程，生命周期组件，容器组件等。基本上将启动过程拆的七零八落，分析的差不多了， 但是还没有从整体的视图下来分析Tomcat 的启动过程。因此，这篇文章的任务就是这个，我们想将Tomcat的启动过程彻底的摸清，把它最后一件衣服扒掉。然后我们就分析连接器和URL请求了，不再留恋这里了。

好吧。我们开始吧。

说到Tomcat的启动，我们都知道，我们每次需要运行tomcat/bin/startup.sh这个脚本，而这个脚本的内容到底是什么呢？我们来看看。

### 1. startup.sh 脚本内容
```shell
#!/bin/sh
os400=false
case "`uname`" in
OS400*) os400=true;;
esac

# resolve links - $0 may be a softlink
PRG="$0"

while [ -h "$PRG" ] ; do
  ls=`ls -ld "$PRG"`
  link=`expr "$ls" : '.*-> \(.*\)$'`
  if expr "$link" : '/.*' > /dev/null; then
    PRG="$link"
  else
    PRG=`dirname "$PRG"`/"$link"
  fi
done

PRGDIR=`dirname "$PRG"`
EXECUTABLE=catalina.sh

# Check that target executable exists
if $os400; then
  # -x will Only work on the os400 if the files are:
  # 1. owned by the user
  # 2. owned by the PRIMARY group of the user
  # this will not work if the user belongs in secondary groups
  eval
else
  if [ ! -x "$PRGDIR"/"$EXECUTABLE" ]; then
    echo "Cannot find $PRGDIR/$EXECUTABLE"
    echo "The file is absent or does not have execute permission"
    echo "This file is needed to run this program"
    exit 1
  fi
fi

exec "$PRGDIR"/"$EXECUTABLE" start "$@"

```
楼主删除了一些无用的注释，我们来看看这脚本。该脚本中有2个重要的变量：
1. PRGDIR：表示当前脚本所在的路径
2. EXECUTABLE：catalina.sh 脚本名称
其中最关键的一行代码就是 `exec "$PRGDIR"/"$EXECUTABLE" start "$@"`，表示执行了脚本catalina.sh，参数是start。

### 2. catalina.sh 脚本实现

然后我们看看catalina.sh 脚本中的实现：
```shell
elif [ "$1" = "start" ] ; then

  if [ ! -z "$CATALINA_PID" ]; then
    if [ -f "$CATALINA_PID" ]; then
      if [ -s "$CATALINA_PID" ]; then
        echo "Existing PID file found during start."
        if [ -r "$CATALINA_PID" ]; then
          PID=`cat "$CATALINA_PID"`
          ps -p $PID >/dev/null 2>&1
          if [ $? -eq 0 ] ; then
            echo "Tomcat appears to still be running with PID $PID. Start aborted."
            echo "If the following process is not a Tomcat process, remove the PID file and try again:"
            ps -f -p $PID
            exit 1
          else
            echo "Removing/clearing stale PID file."
            rm -f "$CATALINA_PID" >/dev/null 2>&1
            if [ $? != 0 ]; then
              if [ -w "$CATALINA_PID" ]; then
                cat /dev/null > "$CATALINA_PID"
              else
                echo "Unable to remove or clear stale PID file. Start aborted."
                exit 1
              fi
            fi
          fi
        else
          echo "Unable to read PID file. Start aborted."
          exit 1
        fi
      else
        rm -f "$CATALINA_PID" >/dev/null 2>&1
        if [ $? != 0 ]; then
          if [ ! -w "$CATALINA_PID" ]; then
            echo "Unable to remove or write to empty PID file. Start aborted."
            exit 1
          fi
        fi
      fi
    fi
  fi

  shift
  touch "$CATALINA_OUT"
  if [ "$1" = "-security" ] ; then
    if [ $have_tty -eq 1 ]; then
      echo "Using Security Manager"
    fi
    shift
    eval $_NOHUP "\"$_RUNJAVA\"" "\"$LOGGING_CONFIG\"" $LOGGING_MANAGER $JAVA_OPTS $CATALINA_OPTS \
      -classpath "\"$CLASSPATH\"" \
      -Djava.security.manager \
      -Djava.security.policy=="\"$CATALINA_BASE/conf/catalina.policy\"" \
      -Dcatalina.base="\"$CATALINA_BASE\"" \
      -Dcatalina.home="\"$CATALINA_HOME\"" \
      -Djava.io.tmpdir="\"$CATALINA_TMPDIR\"" \
      org.apache.catalina.startup.Bootstrap "$@" start \
      >> "$CATALINA_OUT" 2>&1 "&"

  else
    eval $_NOHUP "\"$_RUNJAVA\"" "\"$LOGGING_CONFIG\"" $LOGGING_MANAGER $JAVA_OPTS $CATALINA_OPTS \
      -classpath "\"$CLASSPATH\"" \
      -Dcatalina.base="\"$CATALINA_BASE\"" \
      -Dcatalina.home="\"$CATALINA_HOME\"" \
      -Djava.io.tmpdir="\"$CATALINA_TMPDIR\"" \
      org.apache.catalina.startup.Bootstrap "$@" start \
      >> "$CATALINA_OUT" 2>&1 "&"

  fi

  if [ ! -z "$CATALINA_PID" ]; then
    echo $! > "$CATALINA_PID"
  fi

  echo "Tomcat started."
```

该脚本很长，但我们只关心我们感兴趣的：如果参数是 `start`， 那么执行这里的逻辑，关键再最后一行执行了 `org.apache.catalina.startup.Bootstrap "$@" start`， 也就是说，执行了我们熟悉的main方法，并且携带了start 参数，那么我们就来看Bootstrap 的main方法是如何实现的。

### 3. Bootstrap.main 方法实现
```java
    public static void main(String args[]) {

        System.err.println("Have fun and Enjoy! cxs");

        // daemon 就是 bootstrap
        if (daemon == null) {
            Bootstrap bootstrap = new Bootstrap();
            try {
                bootstrap.init();
            } catch (Throwable t) {
                handleThrowable(t);
                t.printStackTrace();
                return;
            }
            daemon = bootstrap;
        } else {
            Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
        }

        try {
            String command = "start";
            if (args.length > 0) {
                command = args[args.length - 1];
            }
            if (command.equals("startd")) {
                args[args.length - 1] = "start";
                daemon.load(args);
                daemon.start();
            }
            else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                daemon.stop();
            }
            else if (command.equals("start")) {
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
            } else if (command.equals("stop")) {
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                daemon.load(args);
                if (null==daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable t) {
            if (t instanceof InvocationTargetException &&
                    t.getCause() != null) {
                t = t.getCause();
            }
            handleThrowable(t);
            t.printStackTrace();
            System.exit(1);
        }
    }
```
我们看看该方法， 首先 ` bootstrap.init()` 的方法用于初始化类加载器，我们已经分析过该方法了，就不再赘述了，然后我们看下面的try块，默认命令行参数是 `start` ，但我们刚刚的脚本传的参数就是 start， 因此进入该if块
```java
   else if (command.equals("start")) {
         daemon.setAwait(true);
         daemon.load(args);
         daemon.start();
```
1. 设置catalina 的 await 属性为true；
2. 运行 catalina 的 load 方法。该方法内部主要逻辑是解析server.xml文件，初始化容器。我们已经再生命周期那篇文章中讲过容器的初始化。
3. 运行 catalina 的 start 方法。也就是启动 tomcat。这个部分我们上次分析了容器启动。但是容器之后的逻辑我们没有分析。今天我们就来看看。

### 4. Catalina.start 方法
```java
 public void start() {
        if (getServer() == null) {
            load();
        }
        if (getServer() == null) {
            log.fatal("Cannot start server. Server instance is not configured.");
            return;
        }
        long t1 = System.nanoTime();
        // Start the new server
        try {
            getServer().start();
        } catch (LifecycleException e) {
            log.fatal(sm.getString("catalina.serverStartFail"), e);
            try {
                getServer().destroy();
            } catch (LifecycleException e1) {
                log.debug("destroy() failed for failed Server ", e1);
            }
            return;
        }
        long t2 = System.nanoTime();
        if(log.isInfoEnabled()) {
            log.info("Server startup in " + ((t2 - t1) / 1000000) + " ms");
        }
        if (useShutdownHook) {
            if (shutdownHook == null) {
                shutdownHook = new CatalinaShutdownHook();
            }
            Runtime.getRuntime().addShutdownHook(shutdownHook);

            // If JULI is being used, disable JULI's shutdown hook since
            // shutdown hooks run in parallel and log messages may be lost
            // if JULI's hook completes before the CatalinaShutdownHook()
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                        false);
            }
        }
        if (await) {
            await();
            stop();
        }
    }
```

该方法我们上次分析到了 `getServer().start()` 这里，也就是容器启动的逻辑，我们不再赘述。
今天我们继续分析下面的逻辑。主要逻辑是：
```java
 if (useShutdownHook) {
            if (shutdownHook == null) {
                shutdownHook = new CatalinaShutdownHook();
            }
            Runtime.getRuntime().addShutdownHook(shutdownHook);

            // If JULI is being used, disable JULI's shutdown hook since
            // shutdown hooks run in parallel and log messages may be lost
            // if JULI's hook completes before the CatalinaShutdownHook()
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                        false);
            }
        }
        if (await) {
            await();
            stop();
        }
```
可以看到是 `Runtime.getRuntime().addShutdownHook(shutdownHook)`方法。那么这个方法的作用是什么呢？JDK 文档是这样说的：
> 注册新的虚拟机来关闭钩子。
只是一个已初始化但尚未启动的线程。虚拟机开始启用其关闭序列时，它会以某种未指定的顺序启动所有已注册的关闭钩子，并让它们同时运行。运行完所有的钩子后，如果已启用退出终结，那么虚拟机接着会运行所有未调用的终结方法。最后，虚拟机会暂停。注意，关闭序列期间会继续运行守护线程，如果通过调用方法来发起关闭序列，那么也会继续运行非守护线程。

简单来说，如果用户的程序出现了bug， 或者使用control + C 关闭了命令行，那么就需要做一些内存清理的工作。该方法就会再虚拟机退出时做清理工作。再ApplicationShutdownHooks 类种维护着一个IdentityHashMap<Thread, Thread>  Map，用于后台清理工作。那么该线程对象的run方法中是什么逻辑呢？我们来看看：

### 5. CatalinaShutdownHook.run 线程方法实现
```java
 protected class CatalinaShutdownHook extends Thread {
        @Override
        public void run() {
            try {
                if (getServer() != null) {
                    Catalina.this.stop();
                }
            } catch (Throwable ex) {
                ExceptionUtils.handleThrowable(ex);
                log.error(sm.getString("catalina.shutdownHookFail"), ex);
            } finally {
                // If JULI is used, shut JULI down *after* the server shuts down
                // so log messages aren't lost
                LogManager logManager = LogManager.getLogManager();
                if (logManager instanceof ClassLoaderLogManager) {
                    ((ClassLoaderLogManager) logManager).shutdown();
                }
            }
        }
    }
```
该线程是Catalina的内部类，方法逻辑是，如果Server容器还存在，就是执行Catalina的stop方法用于停止容器。（为什么要用Catalina.this.stop 呢？因为它继承了Thread，而Thread也有一个stop方法，因此需要显式的指定该方法）最后关闭日志管理器。我们看看stop方法的实现：

### 6. Catalina.stop 方法实现：
```java
public void stop() {

        try {
            // Remove the ShutdownHook first so that server.stop()
            // doesn't get invoked twice
            if (useShutdownHook) {
                Runtime.getRuntime().removeShutdownHook(shutdownHook);

                // If JULI is being used, re-enable JULI's shutdown to ensure
                // log messages are not lost
                LogManager logManager = LogManager.getLogManager();
                if (logManager instanceof ClassLoaderLogManager) {
                    ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                            true);
                }
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This will fail on JDK 1.2. Ignoring, as Tomcat can run
            // fine without the shutdown hook.
        }

        // Shut down the server
        try {
            Server s = getServer();
            LifecycleState state = s.getState();
            if (LifecycleState.STOPPING_PREP.compareTo(state) <= 0
                    && LifecycleState.DESTROYED.compareTo(state) >= 0) {
                // Nothing to do. stop() was already called
            } else {
                s.stop();
                s.destroy();
            }
        } catch (LifecycleException e) {
            log.error("Catalina.stop", e);
        }
    }
```

该方法首先移除关闭钩子，为什么要移除呢，因为他的任务已经完成了。然后设置useShutdownHook 为true。最后执行Server的stop方法，Server的stop方法基本和init方法和start方法一样，都是使用父类的模板方法，首先出发事件，然后调用stopInternal，该方法内部循环停止子容器，子容器递归停止，和我们之前的逻辑一致，不再赘述。destroy方法同理。

好了，我们已经看清了关闭钩子的逻辑，其实就是开辟一个守护线程交给虚拟机，然后虚拟机在某些异常情况（比如System.exit(0)）前执行停止容器的逻辑。

好。我们回到start方法。

### 7. 回到 Catalina.start 方法

在设置好关闭钩子后，tomcat 的启动过程还没有启动完毕，接下来的逻辑式什么呢？
```java
        if (useShutdownHook) {
            if (shutdownHook == null) {
                shutdownHook = new CatalinaShutdownHook();
            }
            Runtime.getRuntime().addShutdownHook(shutdownHook);

            // If JULI is being used, disable JULI's shutdown hook since
            // shutdown hooks run in parallel and log messages may be lost
            // if JULI's hook completes before the CatalinaShutdownHook()
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                        false);
            }
        }

        if (await) {
            await();
            stop();
        }
```
在设置完关闭钩子之后，会将 useShutdownHook 这个变量为false，然后执行 await 方法。然后执行stop方法，我们记得stop方法式关闭容器的方法，神经病啊，好不容易启动了，为什么又要关闭呢？ 先不着急，我们还是看看 await 方法吧,该方法调用了Server.await 方法，我们来看看：

### 8. Catalian.await 方法实现

注意：该方法很长
```java
    @Override
    public void await() {
        // Negative values - don't wait on port - tomcat is embedded or we just don't like ports
        if( port == -2 ) {
            // undocumented yet - for embedding apps that are around, alive.
            return;
        }
        if( port==-1 ) {
            try {
                awaitThread = Thread.currentThread();
                while(!stopAwait) {
                    try {
                        Thread.sleep( 10000 );
                    } catch( InterruptedException ex ) {
                        // continue and check the flag
                    }
                }
            } finally {
                awaitThread = null;
            }
            return;
        }

        // Set up a server socket to wait on
        try {
            awaitSocket = new ServerSocket(port, 1,
                    InetAddress.getByName(address));
        } catch (IOException e) {
            log.error("StandardServer.await: create[" + address
                               + ":" + port
                               + "]: ", e);
            return;
        }

        try {
            awaitThread = Thread.currentThread();

            // Loop waiting for a connection and a valid command
            while (!stopAwait) {
                ServerSocket serverSocket = awaitSocket;
                if (serverSocket == null) {
                    break;
                }
    
                // Wait for the next connection
                Socket socket = null;
                StringBuilder command = new StringBuilder();
                try {
                    InputStream stream;
                    try {
                        socket = serverSocket.accept();
                        socket.setSoTimeout(10 * 1000);  // Ten seconds
                        stream = socket.getInputStream();
                    } catch (AccessControlException ace) {
                        log.warn("StandardServer.accept security exception: "
                                + ace.getMessage(), ace);
                        continue;
                    } catch (IOException e) {
                        if (stopAwait) {
                            // Wait was aborted with socket.close()
                            break;
                        }
                        log.error("StandardServer.await: accept: ", e);
                        break;
                    }

                    // Read a set of characters from the socket
                    int expected = 1024; // Cut off to avoid DoS attack
                    while (expected < shutdown.length()) {
                        if (random == null)
                            random = new Random();
                        expected += (random.nextInt() % 1024);
                    }
                    while (expected > 0) {
                        int ch = -1;
                        try {
                            ch = stream.read();
                        } catch (IOException e) {
                            log.warn("StandardServer.await: read: ", e);
                            ch = -1;
                        }
                        if (ch < 32)  // Control character or EOF terminates loop
                            break;
                        command.append((char) ch);
                        expected--;
                    }
                } finally {
                    // Close the socket now that we are done with it
                    try {
                        if (socket != null) {
                            socket.close();
                        }
                    } catch (IOException e) {
                        // Ignore
                    }
                }

                // Match against our command string
                boolean match = command.toString().equals(shutdown);
                if (match) {
                    log.info(sm.getString("standardServer.shutdownViaPort"));
                    break;
                } else
                    log.warn("StandardServer.await: Invalid command '"
                            + command.toString() + "' received");
            }
        } finally {
            ServerSocket serverSocket = awaitSocket;
            awaitThread = null;
            awaitSocket = null;

            // Close the server socket and return
            if (serverSocket != null) {
                try {
                    serverSocket.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
        }
    }

```

我们看一下他的逻辑：首先创建一个socketServer 链接，然后循环等待消息。如果发过来的消息为字符串`SHUTDOWN`, 那么就break，停止循环，关闭socket。否则永不停歇。回到我们刚刚的疑问，await 方法后面执行 stop 方法，现在一看就合情合理了，只要不发出关闭命令，则不会执行stop方法，否则则继续执行关闭方法。

到现在，Tomcat 的整体启动过程我们已经了然于胸了，总结一下就是：
1. 初始化类加载器。
2. 初始化容器并注册到JMX后启动容器。
3. 设置关闭钩子。
4. 循环等待关闭命令。

等一下。好像缺了点什么？？？ Tomcat 启动后就只接受关闭命令，接受的http请求怎么处理，还要不要做一个合格的服务器了？？？ 别急，实际上，这个是主线程，负责生命周期等事情。处理Http请求的线程在初始化容器和启动容器的时候由子容器做了，这块的逻辑我们下次再讲。大家不要疑惑。

### 9. 我们知道了Tomcat 是怎么启动的，那么是怎么关闭的呢？

顺便说说关闭的逻辑：

shutdown.sh 脚本同样会调用 Bootstrap的main 方法，不同是传递 stop参数， 我们看看如果传递stop参数会怎么样：
```java
ry {
            String command = "start";
            if (args.length > 0) {
                command = args[args.length - 1];
            }
            if (command.equals("startd")) {
                args[args.length - 1] = "start";
                daemon.load(args);
                daemon.start();
            }
            else if (command.equals("stopd")) {
                args[args.length - 1] = "stop";
                daemon.stop();
            }
            else if (command.equals("start")) {
                daemon.setAwait(true);
                daemon.load(args);
                daemon.start();
            } else if (command.equals("stop")) {
                daemon.stopServer(args);
            } else if (command.equals("configtest")) {
                daemon.load(args);
                if (null==daemon.getServer()) {
                    System.exit(1);
                }
                System.exit(0);
            } else {
                log.warn("Bootstrap: command \"" + command + "\" does not exist.");
            }
        } catch (Throwable t) {
```
可以看到调用的是 stopServer 方法，实际上就是 Catalina的stopServer 方法，我们看看该方法实现：

### 10. Catalina.stopServer 方法

```java
 public void stopServer(String[] arguments) {

        if (arguments != null) {
            arguments(arguments);
        }

        Server s = getServer();
        if( s == null ) {
            // Create and execute our Digester
            Digester digester = createStopDigester();
            digester.setClassLoader(Thread.currentThread().getContextClassLoader());
            File file = configFile();
            FileInputStream fis = null;
            try {
                InputSource is =
                    new InputSource(file.toURI().toURL().toString());
                fis = new FileInputStream(file);
                is.setByteStream(fis);
                digester.push(this);
                digester.parse(is);
            } catch (Exception e) {
                log.error("Catalina.stop: ", e);
                System.exit(1);
            } finally {
                if (fis != null) {
                    try {
                        fis.close();
                    } catch (IOException e) {
                        // Ignore
                    }
                }
            }
        } else {
            // Server object already present. Must be running as a service
            try {
                s.stop();
            } catch (LifecycleException e) {
                log.error("Catalina.stop: ", e);
            }
            return;
        }

        // Stop the existing server
        s = getServer();
        if (s.getPort()>0) {
            Socket socket = null;
            OutputStream stream = null;
            try {
                socket = new Socket(s.getAddress(), s.getPort());
                stream = socket.getOutputStream();
                String shutdown = s.getShutdown();
                for (int i = 0; i < shutdown.length(); i++) {
                    stream.write(shutdown.charAt(i));
                }
                stream.flush();
            } catch (ConnectException ce) {
                log.error(sm.getString("catalina.stopServer.connectException",
                                       s.getAddress(),
                                       String.valueOf(s.getPort())));
                log.error("Catalina.stop: ", ce);
                System.exit(1);
            } catch (IOException e) {
                log.error("Catalina.stop: ", e);
                System.exit(1);
            } finally {
                if (stream != null) {
                    try {
                        stream.close();
                    } catch (IOException e) {
                        // Ignore
                    }
                }
                if (socket != null) {
                    try {
                        socket.close();
                    } catch (IOException e) {
                        // Ignore
                    }
                }
            }
        } else {
            log.error(sm.getString("catalina.stopServer"));
            System.exit(1);
        }
    }

```

注意，该停止命令的虚拟机和启动的虚拟机不是一个虚拟机，因此，没有初始化 Server , 进入 IF 块，解析 server.xml 文件，获取文件中端口，用以创建Socket。然后像启动服务器发送 `SHUTDOWN` 命令，关闭启动服务器，启动服务器退出刚刚的循环，执行后面的 stop 方法，最后退出虚拟机，就是这么简单。

### 11. 总结

我们从整体上解析了Tomcat的启动和关闭过程，发现不是很难，为什么？因为我们之前已经分析过很多遍了，有些逻辑我们已经清除了，这次分析只是来扫尾。复杂的Tomcat的启动过程我们基本就分析完了。我们知道了启动和关闭都依赖Socket。只是我们惊奇的发现他的关闭竟然是如此实现。很牛逼。我原以为会像我们平时一样，直接kill。哈哈哈。

好吧。今天我们就到这里 ，tomcat 这座大山我们已经啃的差不多了，还剩一个 URL 请求过程和连接器，这两个部分是高度关联的，因此，楼主也会将他们放在一起分析。透过源码看真相。

连接器，等着我们来撕开你的衣服！！！！

good luck ！！！！































































