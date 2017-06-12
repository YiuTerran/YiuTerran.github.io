# CometD 源码学习[0]

tags：[cometd,jetty,java]

---

## jetty生命周期
Jetty的核心组件是`Server`和`Connector`，`Server`基于`Handler`容器工作，里面包括`ServletHandler`，`SessionHandler`等处理器，`Server`本身也继承自`Handler`类。`Connector`类用于监听连接请求；此外还有`Container`用来管理`MBean`。
Jetty的Server扩展是通过实现`Handler`并将至注册到`Server`中来实现的。
整个Jetty的组件的生命周期管理是基于观察者模板设计的，每个组件都有个`Listener`，用来监听Jetty启动/停止过程中的事件。

### Handler
Jetty本身提供了两类`HandlerWrapper`和`ScopeHandler`两种`Handler`，前者是一个装饰器修饰的`Handler`，用来做委托(`Proxy`模式)，后者是一个拦截器——在调用`Handler`之前或之后做某些事情。

### prestart
Jetty在启动之前会先初始化jetty的相关配置(start.ini)，然后通过自己的`IOC`(`XmlConfiguration`)将这些服务组装在一起，最后调用`start`启动这些组件。其中最重要的配置文件包括`jetty.xml`, `jetty-deploy.xml`以及`contexts/*.xml`。然后根据配置文件中的参数新建一个进程应用JVM参数（如果有`--exec`，没有的话不会再起新的进程，`start.ini`中的JVM参数就不可能重新得到应用)。

### 启动
Jetty的启动入口是`Server`类或者其子类。下图是Jetty启动过程：
![Jetty Start process][1]
在创建线程池后，Server开始依次调用已注册Handler组件的`Start`方法，直至整个调用链结束（调用链的末尾应该是用户自定义的service），这一步是初始化各组件（filter, servlet，包括用户的服务）的配置；然后启动`Container`中已注册的MBean（for JMX），最后启动`Connector`开始接受请求。

Jetty作为一个轻量级web容器，不仅可以接受http协议作为web服务器，还可以与其他web应用服务器集成（如Jboss或Apache），这时候Jetty工作于AJP协议。
### web服务器
对于http协议，按着传统的划分方式，分为BIO（阻塞式）和NIO（非阻塞式），以及AIO（异步式），windows的IOCP是AIO。

>BIO是一个连接一个线程。
NIO是一个请求一个线程。
AIO是一个有效请求一个线程。
#### BIO
如果jetty工作在BIO模式（选用`org.eclipse.jetty.server.bi.SocketConnector`作为`Connector`），建立连接的步骤分为：
1. 创建队列线程池，用于处理请求；
2. 创建ServerSocket用于准备接受请求；
3. 创建一个或多个监听线程（Accptor)，开始监听。
4. 对于每个连接，BIO从线程池中分配一个线程进行处理；
时序图如下：
![Connect process][2]

Accptor对每个请求创建ConnectorEndPoint，后者对实际消息做出具体解析并应答。Jetty9 移除了BIO的Connecter（现在是异步的世界了…）
如果工作在AJP协议下，与 HTTP 方式唯一不同的地方的就是将 SocketConnector 类替换成了 Ajp13SocketConnector，即监听的协议不同而已。

#### NIO
NIO是非阻塞的，类似于linux中的`select`系统接口：
1. 创建N个`Acceptor`对象，每个对应一个`SelectSet`对象，用于存放已注册的socket集合；
2. 创建N个`Selector`线程用于轮询`SelectSet`(`select`)或监听`SelectSet`中的事件(`epoll`)，线程数同`Acceptor`的个数(`jetty.xml`中指定）；
3. `Connector`接受(`accept`)到请求，得到`Socket`，将之设为非阻塞，然后以轮询机制分发给`Acceptor`并返回（异步)；
4. `Acceptor`将之放入自己的`SelectSet`，并返回；
5. `Selector`检测到新的`Socket`，开始监听该`Socket`的`read`事件（`select`）；
6. 一旦有新事件到来，立刻新建`ConnectorEndPoint`，调用`schedule`方法并返回继续监听该`Socket`（异步）；
7. `schedule`方法调用线程池中的线程，进行实际的逻辑处理(`worker`)，该线程会调用`Server`的`handle`方法，这里形成`handle`的调用链（这是在server启动前注册到server中的）。

NIO的一般工作原理用代码描述如下：
```java
 Selector selector = Selector.open();   //实例化selector
 ServerSocketChannel ssc = ServerSocketChannel.open(); //实例化socket
 ssc.configureBlocking( false );  //非阻塞
 SelectionKey key = ssc.register( selector, SelectionKey.OP_ACCEPT ); //注册事件
 ServerSocketChannel ss = (ServerSocketChannel)key.channel();
 SocketChannel sc = ss.accept();
 sc.configureBlocking( false );
 SelectionKey newKey = sc.register( selector, SelectionKey.OP_READ );
 Set selectedKeys = selector.selectedKeys();
```
### AIO
java7开始支持AIO，而jetty则从jetty9开始支持这一特性。AIO在linux上使用`epoll`完成，在windows上则使用IOCP（IO完成端口）完成。

一般而言，AIO对应`Proactor`模式，NIO对应`Reactor`模式，两者最大的区别在于IO是由谁来完成：AIO中，由内核完成IO，然后将结果通知给用户（信号/回调函数）；NIO中，内核只是将准备好进行IO的描述符通知给用户（信号/回调函数），然后由用户自己处理IO。

---
需要了解的预备知识到此结束，下一篇开始正式分析CometD源码。

  [1]: http://www.ibm.com/developerworks/cn/java/j-lo-jetty/image011.jpg
  [2]: http://www.ibm.com/developerworks/cn/java/j-lo-jetty/image013.jpg