# 源码目录结构介绍&Remoting通信层

##一：源码目录结构介绍
RocketMQ源码分为以下几个package：

 - `rocketmq-broker`：整个mq的核心，他能够接受producer和consumer的请求，并调用store层服务对消息进行处理。**HA服务**的基本单元，支持**同步双写**，**异步双写**等模式。
 - `rocketmq-clien`：：mq客户端实现，目前官方仅仅开源了java版本的**mq客户端**，c++，go客户端有社区开源贡献。
 - `rocketmq-common`：一些模块间通用的**功能类**，比如一些配置文件、常量。
 - `rocketmq-example`：官方提供的**例子**，对典型的功能比如order message，push consumer，pull consumer的用法进行了示范。
 - `rocketmq-filtersrv`：消息**过滤**服务，相当于在broker和consumer中间加入了一个filter代理。
 - `rocketmq-remoting`：基于netty的底层**通信实现**，所有服务间的交互都基于此模块。
 - `rocketmq-srvut`：解析命令行的工具类。
 - ``rocketmq-store`：存储层实现，同时包括了**索引**服务，高可用**HA**服务实现。
 - `rocketmq-tools`：mq**集群管理**工具，提供了消息查询等功能。


----------
##二：rocketmq-remoting通信层介绍
remoting模块是mq的基础通信模块，理解通信层的原理对理解模块间的交互很有帮助。底层基于Netty网络库驱动，因此需要先了解一些基本的Netty原理。
###1、Netty基本知识
![nettyThread](netty_thread_module.png) <br>
首先，我们需要了解Netty的线程模型：Netty运用了**reactor**模式，采用了**监听线程池**和**IO线程池分离**的思想，数据的流转在Netty中采取了类似**职责链**的设计模式，因此数据看起来就像在管道中流动一样了。<br>
现在我们只需要知道，我们能够定义自己的handler并插入管道即可实现对数据的操作了。目前大致了解即可，稍后我们会结合mq的代码讲解。


----------


###2、mq通信协议
####(1) 消息格式
 ![proto](proto.png) <br>
  重点关注一下header字段，他有2种编码方式，一种**JSON**格式，另一种是ROCKETMQ格式。重点关注JSON格式：
  <br>
![header](protp_header.png) <br>
这里直接引用了官方文档里的图片，`RequestCode.java`和`ResponseCode.java`文件包含了所有的操作码，推荐调试2个模块之间的通信的时候可以以操作码为索引。一个实际的请求如图：
<br>
![header_actual](header.png) <br>
code=103表示他是一个`REGISTER_BROKER`消息
<br>
####(2) mq的消息处理逻辑
那么，对于一个实际的请求，mq是如何进行编解码以及分发请求的呢？比较重要的两个类包括NettyRemotingClient和NettyRemotingServer，这里以NettyRemotingServer为例子先看它的启动：
![remotingServer_start](remotingServer_start.png) <br>
可以看到ch.pipeline().addLast就是往管道里添加数据的处理逻辑，首先需要知道对于每一个事件处理器handler，他可以处理的事件包括了以下几种(覆盖父类方法即可实现)，只要满足条件数据会经过每一个handler对应的事件处理方法：

 - `channelActive`、`channelInactive`：**连接建立**和**连接关闭**的时候会被回调。
 - `channelRead`：当channel有数据**可读**时会回调到这个函数。mq正是从这个函数将请求分发到后端线程进行处理的。
 - `exceptionCaught`：发生**异常**时回调。
 - `userEventTriggered`：当上面的事件都不满足自己的需求时，用户可以在这里面**自定义**的事件处理方法。<br>
<br>
_现在回头来看，mq的**pipeline管道**定义如下handler的含义：_
 - `NettyEncoder`、`NettyDecoder`：mq对应的编码器和解码器的逻辑，他们分别覆盖了父类的**encode**和**decode**方法。
 -  `IdleStateHandler`：Netty自带的心跳管理器
 - `NettyConnetManageHandle`：连接管理器，他负责捕获新连接、连接断开、异常等事件，然后统一调度到NettyEventExecuter处理器处理。
 - `NettyServerHandler`：当一个消息经过前面的解码等步骤后，然后调度到channelRead0方法，然后根据消息类型进行分发
 <br>
 继续跟踪`NettyServerHandler`代码：
![delegate](delegate_message.png) <br>
 <br>
 _接下去的代码的处理逻辑分为处理消息请求和消息响应。<br>_
**(a)处理消息请求processRequestCommand**<br>
首先看`NettyRemotingAbstract`类中的一个成员：
 ```
HashMap<Integer/* request code */, Pair<NettyRequestProcessor, ExecutorService>> processorTable
 ```
可以看到注释，键表示了`request code`，mq中可以为不同类型的请求码指定不同的处理器`Processor`处理，但是要注意消息实际的处理并不是在当前线程，而是被封装成task放到`Processor`对应的线程池处理：<br>

 ```
final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
pair.getObject2().submit(requestTask);
 ```
 在RocketMQ中能看到很多地方都是这样的处理，这样的设计能够最大程度的保证**异步**，保证每个线程都专注处理自己负责的东西。 以下是`Processor`的实现：<br>
![processer-implement](processer_implement.png) <br>
<br>
最后，`processRequestCommand`这个函数的整体处理逻辑如下所示：<br>
![flow](flow1.png) <br>

 另外，要注意一下，第二步构建task的时候，运用了**模板设计模式**，在任务的执行前后加入了一个hook：我们可以利用这个hook进行一些额外的操作，比如消息的加密解密。

 ```
rpcHook.doBeforeRequest(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
Processor.processRequest()
rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
```
<br>
**(b)处理消息响应processResponseCommand**<br>
实际上，这部分的处理并不是太难，首先理解下面这个结构：

```
protected final ConcurrentHashMap<Integer /* opaque */, ResponseFuture> responseTable
```
opaque表示请求发起连接方在同个连接上不同的请求标识代码，每次发送一个消息的时候，他可以选择同步阻塞的方式和异步非阻塞的方式，不管是哪种方式，他都会保存操作码到ResponseFuture的映射。
重点讲解一下`ResponseFuture`这个类，这个类中比较重要的成员包括一个回调函数`invokeCallback`，以及一个信号量`semaphore`。

 - 对于**同步消息**，这二个参数通常是个null。
 - 对于**异步消息**，`invokeCallback`的作用就是在收到消息响应的时候能够根据`responseTable`找到操作码对应的回调函数；`semaphore`的主要作用是用作**流控**，当多个线程同时往一个连接写数据时可以通过信号量控制permit同时写许可的数量。<br>
简单来说，总体流程如下：<br>
![flow](flow2.png) <br>
当然，流程图未列举的操作还包括释放信号量资源，以及清空`responseTable`表相关键值对信息等操作。<br>
<br>
`NettyRemotingClient`的处理实际上与·NettyRemotingServer·的处理基本一致，唯一不同的是Netty pipeline中**连接管理**相关的handler额外还处理了`connect事件`，该事件在客户端主动连接对端成功后回调。












