# cometd 学习文档

tags: [push,java,cometd]

---

1. cometd client 分为remote client和local client，remote client即是传统的cometd客户端，分为java和javascript两个版本；当remote client与server建立bayeux协议连接后，server session才被创建；而local client则是server侧的client，当server想要创建一个不与remote client直接连接的session时，必须先创建一个local client；
2. local client的作用在于，如果server并非简单的无条件转发信息到其他remote client，而是想要做一些server端的处理（比如仅仅发往特定的客户端），就需要一个local client。service通道都对应一个local client；
3. 通道类型分为meta channel, broadcast channel和service channel，meta channel是禁止订阅的，用来处理协议的基本功能，包括/meta/handshake, /meta/connect, /meta/disconnect, /meta/subscribe, /meta/unsubscribe, /meta/publish, /meta/unsucessful；而broadcast channel类似与聊天室，服务器无条件转发来自broadcast channel的信息到所有的subscribers；service channel则类似于私聊，服务器有条件转发到指定的订阅了该服务的某一个或多个remote client；
4. server端是一个`BayeuxServer`对象，相关联的对象包括：
    * transport，包括http和websocket，或其他自己实现的接口，这里用来实现bayeux的底层通信方式；
    * channels，通道
    * extensions，扩展用来与bayeux协议交互，是一种listener，可以用于在消息接收后、发送前等时刻对消息进行自定义处理
    * authorization，认证机制，一般通过`SecurityPolicy`来实现；也可以通过`Authorizer`进行更细粒度（如针对某个channel）的处理；
    * 消息处理，可以通过回调（监听）对客户端发来的消息进行处理；
5. Listener.
    * 客户端对某个channel的listener，用来处理服务器发送（或转发）的message(通过`ClientSession.getChannel(String).addListener(ClientSessionChannel.MessageListener)`来添加)；此外也可以通过添加extension对消息做最初或最后的处理；
    * Server端的Listener类似，但更加丰富，包括：
        * extension, 对server的或对session的
        * channel create|destroy listener
        * subscribe|unsubscribe channel
        * session被remove
        * 与server的message queue交互(`MaxQueueListener`, `DeQueueListener`等)
        * MessageListener

6. 消息流动的过程：
    1. 客户端通过client-side的channel发送消息，这些消息首先通过clientsession extension，做最后的过滤；然后通过底层的transport发往server。transport会将message转换为JSON格式，server transport接受到这些消息后，再将JSON字符串转为普通消息；
    2. 消息在server侧首先通过server extension，做最初的处理，此时如果被拒绝，返回给client的错误标示该消息已被删除；
    3. 消息随即被送往serversession extension，如果被拒绝，操作同上；
    4. 消息被送往security policy和authorizers接受审核，如果被拒绝，返回给客户端说明未通过认证；
    5. 消息被送往channel listener，服务器可以在channel listener里面对消息做任意修改；该步骤后，消息被冻结；
    6. 如果是nonbroadcast channel，会随即回复一条消息给发送者，表明服务器已收到消息并且该消息不是广播消息（貌似是将消息原样返回）；
    7. 如果是broadcast message，消息会通过serversession extension，然后插入server的message queue，准备发出；
    8. 如果是lazy message，消息需要等待一段时间，否则消息被立即发送出去；如果消息被发往remote client，会单独开一个线程进行异步发送，后续步骤包括转换为JSON格式，通过底层的transport等，类似客户端发送消息之前的方式；如果是local client，就直接发送过去；
    9. 不管是不是广播消息，消息被插入消息队列后，服务器会返回一条信息给发送者，标明发送结束；
    10. 客户端接收到消息，底层的tranport将消息转回普通格式，然后依次经过client session extension, channel listener和channel subscriber；
    11. server每收到一条bayeux消息，会分配一个单独的线程进行消息处理，所有的listener都在这个单独的线程中被依序唤醒。client与server之间的连接是有限的，如果同一个连接在短时间内收到大量消息，而消息的处理过程过慢，就会导致后续消息被堵塞而无法得到及时处理。因此服务器在处理消息过程中如果有**耗时处理**，务必要**单开线程**；
7. 消息类型：
    * 客户端发往server的，主要是meta消息；
    * 客户端发往指定客户端的，需要通过service通道，然后通过clientId deliver过去
    * 服务器发往客户端的，需要服务器本地起一个local client session，与服务器握手，然后在该通道publish一个消息，这个local client充当了消息的sender
    * 即使client没有subscribe一个channel，也可以在这个channel上publish message
8. cometd server无法保证消息被正确送往client，除非client和server都打开message acknowledgment extension，这样二者在handshake时会在ext段增加特殊属性，协商完毕后，客户端会对每一条接受到的消息做出应答；这种机制提供了一种**不可靠的**失败重传功能；
9. seti是oort集群时用来发送消息的一种工具，如果a想向b发送信息，但是二者连在不同的comet节点上，这时候就需要转发消息、操作clientId等。seti可以将client与其他标识进行关联，这种关联可以是一对一的，也可以是一对多的。比如说，一个userid有多个deviceid，我们可以将userid与该用户的所有device的clientId都关联起来，也可以将其deviceid和对应的clientid对应起来，这样就可以直接转发了。
10. 每个oort节点仅有1个seti对象，当userid第一次与seti关联时，会向所有节点广播此消息，这样其他节点都知道如果需要向某个userid发消息，需要把消息转发给这个seti；用户断开连接或者超时被移除session，这种关联会自动取消。当一个seti中，userid关联的所有session都断开连接时，userid会解除与seti的关联，然后广播此消息。我们可以监听这两种广播消息。
11. 通过seti发送消息很简单，我们只需要指定userid，seti会自动找到其所连接的comet节点，进行发送；
12. comet节点之间的数据共享。`OortObject`用于解决此问题，这是一个分布式的架构，所有节点拥有此对象，此对象存放所有节点的共享数据，每个节点只能修改属于自己的那一部分，其他的是只读的，每次修改会广播给其他节点，其他节点可以监听这种广播，并做一些自定义修改。
13. 服务转发功能，如果集群各个节点提供的服务不一致，可能需要此功能用于转发服务。