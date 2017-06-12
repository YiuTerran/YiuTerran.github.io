# CometD源码学习[1]
tags：[cometd,jetty,java]

---
首先学习server部分，主要包括`cometd-java-server`这个package，同时涉及到`cometd-java-common`和`bayeux-api`这两个package。

## org.cometd.server.CometDServlet
在web.xml中，服务的配置顺序一般是`CometDServlet`，`oort`，`seti`和用户自定义应用的`Servlet`，我们也按这个顺序来看。显然，这个类主要用于Long-Polling模式。
`init`中主要就是新建（如果未导出）并启动一个bayeuxServer;
`service`中，"OPTION"请求，用于允许CORS访问，直接返回200；否则转发给transport；
`destroy`用于生命周期中stop过程调用，依次cancelSchedule, stop bayeuxServer, remove导出的bayeuxServer.

## org.cometd.server.BayeuxServer
这个接口规定了bayeux服务器需要实现的接口。值得关注的地方：
1. 可监听事件：
    a. ChannelListener用于监听add/remove Channel的事件；
    b. SessionListener用于监听add/remove Session的事件，这个比较重要。可以通过Session建立事件来给client push 欢迎消息，通过Session removed事件来确认client断开连接。
    c. SubscriptionListener用于监听订阅事件；
2. extension接口:
    extension本质是一个消息钩子，可以在rcv之初/send之末对消息做一些修改（主要是操作bayeux协议的`ext`字段），所以参数中ServerMessage都是Mutable的.这里normal message和meta message被区分开来。

## org.cometd.server.BayeuxServerImpl
顾名思义，bayeuxServer的实现类。
`VALID`用了一个技巧，即字符本质上是一个short的ASCII码；
`System.identityHashCode`用于获取对象的原始hashcode码；
`SecureRandom`是一个强加密的随机数类；
`listeners`, `extension`都被存放在线程安全的`CopyOnWriteArrayList`里面；
client_id与`ServerSessionImpl`、channel_name与`ServerChannelImpl`的映射也被存放在线程安全的`ConcurrentHashMap`里面；
Server支持的transport被存入`LinkedHashMap`里面，因为transport的顺序很重要，优先使用迭代中最前的，不可行时才使用后续者；
`currentTransport`是一个`ThreadLocal`变量，因为每个线程（连接）当前的Transport肯定不一样。

_scheduler是一个周期性定时器，此外是一个policy、一个JSON的server，这三个变量。

### dostart
1. 首先初始化Meta Channel：创建Channel并增加相关的Listener；
2. 初始化JSON服务器，这里有一个缺省的实现（`JettyJSONContextServer`），但是用户也可以通过option自定义一个实现类（通过反射，使用`isAssignableFrom`判断是否是`JSONContext.Server`的子类）；
3. 初始化transport.如果没有设置，初始化为websocket接口，否则依序初始化配置文件中指定的端口并添加到容器中；如果`allowedTransports`没有设置，默认允许所有transport，否则依序初始化配置文件中允许的且存在于transport列表中的transport；上述所有数据都被添加到公用的容器中了。
4. 启动_scheduler，执行周期性扫除(sweep)任务（每次计时任务结束后需要手动再启动定时任务），该任务会扫描所有Channel和端口以及session，扫描周期默认是997ms，可以自己设置。
session扫描即服务端超时机制，如果now>一定时间间隔，则从服务端移除session；Channel扫描就是检测Channel的订阅数量，如果没有活动的（即已握手的session的）订阅，那么就从BayeuxServer中清除（除非设置为`persistent`）；
5. 最后是2.9新增的`validateMessageFields`用于校验消息格式。

### createChannelIfAbsent
创建通道，传入channel名和初始化器。
如果Channel name尚不存在：
1. 根据channel name创建`ChannelId`，然后创建一个新的`ServerChannelImpl`，后者是一个`ServerChannel`接口的实现类。注意在`ServerChannelImpl`的构造函数中，如果非broadcast channel，会被设置为persistent的；
2. 存放channel，会再次检测channel是不是已经存在（多线程检测），确认无误后，开始配置channel；使用传入的`initializers`和已注册的`listener`配置channel；
3. 初始化完毕，触发ChannelListener的Channel added事件；
如果Channel name已存在：
什么都不做，简单的给将channel的存活评估值（_sweeperPasses)重置。会再次check channel（`putIfAbsent`）是不是已存在于容器中。

## PushServlet
先跳过Oort和Seti，直接看PushServlet（我们的应用程序）。
由于在CometDServlet里面已经导出了`bayeuxServer`，这里可以通过`getServletContext()`直接拿到Server了.
现在可以创建`SecurityPolicy`和`PushService`了。

## AbstractService
顾名思义，这个类是`abstract`的，注意构造函数里面会先create一个`LocalSession`，然后自己和自己握手。这个LocalSession本子上是用于服务端主动publish消息的.
**初始化的时候可以指定线程池的大小，否则使用同步访问。**显然，如果不使用线程池，那么在处理消息时如果有费时间的操作，必须新建线程。
这里用了一个技术，使用反射技术查看自己所处的类的`Modifier`是不是`public`的。

### addService
创建Channel后，正常流程走到这里。传入channel name和callback func name，利用反射技术进行映射。
`getClass`拿到类，然后从当前开始逐层向上遍历直到`AbstractService`，执行以下操作：
`getDeclaredMethods`拿到方法集合，遍历方法集合，判定名称相等且有`public`描述符，那么找到候选`Method`。
候选`Method`的参数必须符合固定的签名类型，这里用`isAssignableFrom`配合`getParameterTypes()`来进行判断；
**注意**：这里会主动调用`createChannelIfAbsent`创建服务（这里就没机会做配置了）；

创建一个`Invoker`，在并行队列里放入`messageName`和`Invoker`的映射。
该`Invoker`被增加为Channel的Listener.

### handle
所有消息首先经过该函数进行分发。
首先验证消息，创建回复并关联。调用接受的extension对消息进行扩展；从消息中取出channel，对消息进行认证(`Authorizer`)；然后依据是否是Meta消息进行不同的分发，无论是接受消息还是发送消息，最终都是调用了`doPublish`方法。

## AbstractService.Invoker
`ServerChannel.MessageListener`接口的实现类。
### OnMessage
消息的观察者。
Server可以通过`isSeeOwnPublish`相关参数的配置控制是否接收自己publish的消息，然后调用(`invoke`）回到函数。如果初始化的时候传递了最大线程数，那么这里就从线程池里面拿线程然后处理消息；**否则直接在当前线程里面处理消息**(`doInvoke`)；
如果回调函数的签名中有返回值，这个值会被立刻返回(send)给client端。

现在再次回到`BayeuxServerImpl`类中，`doStart`会给所有的`Meta` Channel增加Listener，这是预置的Meta handler；按着Bayeux协议约定的过程，client会首先handshake。

## BayeuxServerImpl.HandshakeHandler
先看父类，`HandlerListener`是`ServerChannel.ServerChannelListener`的实现类。
`isSessionUnknown`没啥好说的；`toChannelList`有个有趣的地方，可以用`Collections.singletonList`生成单元素列表），对某些需要传一个集合，但是实际上只要传一个元素的API很有用；

### OnMessage
handshake的时候session理论上还不存在，于是新建一个`ServerSession`，并且将HTTP的相关报头转移过来（UA），然后得到关联(`getAssociated`)的消息实体（消息可能是捎带回去的）。
这时候先判断`SecurityPolicy`有没有设置，如果设置了，就先判断`canHandshake`是不是成立，如果不成立，就要`reply.setSuccessful(false)`，并设置错误原因。

这里有个问题，reply并没有传给`canHandshake`，但是`message`被传进去了。根据`ServerMessageImpl`的实现，`canHandshake`也可以用`message.getAssociated`里面拿到reply，然后增加想要的字段（ext）。代码显示，这里并没有往advice里面添加重连间隔（`interval`)字段。
【这里显示了nest class如何得到outer class的实例，直接用`BayeuxServerImpl.this`.】

一切正常，`ServerSession`首先和自己握手，增加`ServerSession`（在此处**通知回调**，即应用程序注册的监听Session添加的方法），定义reply中的某些字段；如果`canHandshake`返回false，则返回403（handshake denied）错误；注意：如果应用程序的监听器没有设置`advice`中的`reconnect`字段，这里默认会填入`none`。

`reconnect`字段分为3种，正常是`retry`；`handshake`一般是402错误，要求重新握手；`none`就是禁止自动重连了。[^1]
## BayeuxServerImpl.ConnectHandler
心跳信息处理。
### OnMessage
如果session未知，那就需要重新握手，返回402错误。否则对session进行续期，并返回advice，内容包括`timeout`和`interval`字段（如果没有设置，默认是不过期的）

## BayeuxServerImpl.SubscribeHandler
订阅Channel处理。
### OnMessage
402同上。如果没有`subscription`字段或者字段不符合格式，403；如果Channel不存在，尝试创建Channel(policy的`canCreate`，以及内部Channel格式约束)，失败则403。

## BayeuxServerImpl.UnsubscribeHandler
退订处理。
### OnMessage
退订只要session存在，消息格式正确，没有理由不让你退XD

## BayeuxServerImpl.DisconnectHandler
断开连接处理。
### OnMessage
session存在即可断开。
会激活移除session的回调（timeout=false）
刷新session=>

## ServerSessionImpl
### flush
刷新session会取消上一次的lazyTask；
### LazyTask
`Runnable`子类，其`scheduler`方法最终调用`BayeuxServerImpl`中的同名方法，作为一个计划任务执行（`Scheduler`是jetty的基础工具类）。
### Setxxx
可以设置的包括：
Interval
timeout
### ServerSessionListener
可以监听的事件包括：
`RemoveListener`=>移除Session时通知
`MessageListener`=>有消息时通知
`QueueListener`=>消息被加入队列时通知
`DeQueueListener`=>消息出列时通知
`MaxQueueListener`=>队列满时通知

[^1]: http://docs.cometd.org/reference/bayeux_message_fields.html