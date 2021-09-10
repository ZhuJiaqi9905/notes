# Server

## 协议

kafka定制了多种协议类型。每个协议类型都有Request和Response。例如：`ProduceRequest`和`ProduceResponse`。主要在`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\common\requests`中。

Request类型包含相同结构的RequestHeader和不同结构的RequestBody。

## TimingWheel

kafka中存在大量的延时操作，因此它实现了一个用于延时功能的定时器。

TimingWheel是一个存储定时任务的环形队列，底层是数组实现的。数组的每个元素可以存放一个TimerTaskList。
TimerTaskList是一个环形的双向链表，其中的每一项都是定时任务项(TimerTaskEntry),其中封装着定时任务TimerTask。

时间轮由多个时间格组成，每个时间格代表当前时间轮的基本时间跨度tickMs。
时间轮的时间格个数是固定的，用wheelSize表示。
一个时间轮能表示的总interval = wheelSize * tickMs。

多个时间轮叠加来表示更长的时间。
比如最短的时间轮的interval是20ms，那么它的上一级时间轮的每个tickMs就是20ms。

时间轮降级：高层时间轮中的事件距离执行时，还差小于tickMs的时间，就把这个事件放到更低级的时间轮中。

Kafka中使用了JDK中的DelayQueue来推进时间轮。具体做法是对于每个使用到的TimerTaskList都加入DelayQueue。
使用一个ExpriedOperationReaper线程来获取DelayQueue中到期的任务列表。

## 延时操作

不是定时操作，在超时时间之前完成就行。

延时生产，延时拉取，延时数据删除。

延时操作创建之后会被加入延时操作管理器(DelayedOperationPurgatory)来做专门的处理。
每个延时操作管理器都会配备一个定时器来做超时管理。
定时器的底层是TimingWheel。
延时操作需要支持外部事件的触发，所以还要配备一个监听池来负责监听每个分区的外部事件。

## Controller 控制器

kafka中有一个或者多个broker,其中一个brocker会被选举为controller，它负责管理整个集群中所有分区和副本的状态。





主程序：Kafka.scala

Kafka.scala里面buildServer()，new一个KafkaServer。添加shutdown的钩子。调用server.startup()

KafkaServer的startup()里面调用了socketServer的startup(startProcessingRequests = false),最后调用了socketServer的startProcessingRequests.

 Acceptor has N Processor threads that each have their own selector and read requests from sockets, M Handler threads that handle requests and produce responses back to the processor threads for writing.



### KafkaServer调用的startup()

```scala
def startup(startProcessingRequests: Boolean = true,
            controlPlaneListener: Option[EndPoint] = config.controlPlaneListener,
            dataPlaneListeners: Seq[EndPoint] = config.dataPlaneListeners): Unit = {
  this.synchronized {
    createControlPlaneAcceptorAndProcessor(controlPlaneListener)
    createDataPlaneAcceptorsAndProcessors(config.numNetworkThreads, dataPlaneListeners)
    if (startProcessingRequests) {
      this.startProcessingRequests()
    }
  }
 ...
}
```

startup()里面好像默认是没有ControlPlane的事的。所以只创建了DataPlane的Acceptor和Processor。

```scala
private def createDataPlaneAcceptorsAndProcessors(dataProcessorsPerListener: Int,
                                                  endpoints: Seq[EndPoint]): Unit = {
  info(s"in createDataPlaneAcceptorsAndProcessors")
  endpoints.foreach { endpoint =>
    connectionQuotas.addListener(config, endpoint.listenerName)
    val dataPlaneAcceptor = createAcceptor(endpoint, DataPlaneMetricPrefix)
    addDataPlaneProcessors(dataPlaneAcceptor, endpoint, dataProcessorsPerListener)
    dataPlaneAcceptors.put(endpoint, dataPlaneAcceptor)
    info(s"Created data-plane acceptor and processors for endpoint : ${endpoint.listenerName}")
  }
  info(s"out createDataPlaneAcceptorsAndProcessors")
}
```

```scala
private def createAcceptor(endPoint: EndPoint, metricPrefix: String) : Acceptor = {
  val sendBufferSize = config.socketSendBufferBytes
  val recvBufferSize = config.socketReceiveBufferBytes
  new Acceptor(endPoint, sendBufferSize, recvBufferSize, nodeId, connectionQuotas, metricPrefix, time)// call OpenServerRdmaChannel to init a serverRdmaChannel
}
```

OpenServerRdmaChannel里面建立了一个serverChannel并bind，listen.

### socketServer的startProcessingRequests()

```shell
def startProcessingRequests(authorizerFutures: Map[Endpoint, CompletableFuture[Void]] = Map.empty): Unit = {
  info("Starting socket server acceptors and processors")
  this.synchronized {
    if (!startedProcessingRequests) {
      startControlPlaneProcessorAndAcceptor(authorizerFutures)
      startDataPlaneProcessorsAndAcceptors(authorizerFutures)
      startedProcessingRequests = true
    } else {
      info("Socket server acceptors and processors already started")
    }
  }
  info("Started socket server acceptors and processors")
}
```

```scala
private def startControlPlaneProcessorAndAcceptor(authorizerFutures: Map[Endpoint, CompletableFuture[Void]]): Unit = {
  info(s"in startControlPlaneProcessorAndAcceptor")
  controlPlaneAcceptorOpt.foreach { controlPlaneAcceptor =>
    val endpoint = config.controlPlaneListener.get
    startAcceptorAndProcessors(ControlPlaneThreadPrefix, endpoint, controlPlaneAcceptor, authorizerFutures)
  }
  info(s"out startControlPlaneProcessorAndAcceptor")
}

```

因为没有ControlPlaneProcessor,所以这个函数什么都没做。



```scala
/**
 * Starts processors of all the data-plane acceptors and all the acceptors of this server.
 *
 * We start inter-broker listener before other listeners. This allows authorization metadata for
 * other listeners to be stored in Kafka topics in this cluster.
 */
private def startDataPlaneProcessorsAndAcceptors(authorizerFutures: Map[Endpoint, CompletableFuture[Void]]): Unit = {
  info(s"in startDataPlaneProcessorsAndAcceptors")
  val interBrokerListener = dataPlaneAcceptors.asScala.keySet
    .find(_.listenerName == config.interBrokerListenerName)
  val orderedAcceptors = interBrokerListener match {
    case Some(interBrokerListener) => List(dataPlaneAcceptors.get(interBrokerListener)) ++
      dataPlaneAcceptors.asScala.filter { case (k, _) => k != interBrokerListener }.values
    case None => dataPlaneAcceptors.asScala.values
  }
  orderedAcceptors.foreach { acceptor =>
    val endpoint = acceptor.endPoint
    startAcceptorAndProcessors(DataPlaneThreadPrefix, endpoint, acceptor, authorizerFutures)
  }
  info(s"out startDataPlaneProcessorsAndAcceptors")
}
```

这个函数最后一步调用了下面的函数

```scala
private def startAcceptorAndProcessors(threadPrefix: String,
                                       endpoint: EndPoint,
                                       acceptor: Acceptor,
                                       authorizerFutures: Map[Endpoint, CompletableFuture[Void]] = Map.empty): Unit = {
  debug(s"Wait for authorizer to complete start up on listener ${endpoint.listenerName}")
  waitForAuthorizerFuture(acceptor, authorizerFutures)
  debug(s"Start processors on listener ${endpoint.listenerName}")
  acceptor.startProcessors(threadPrefix)// start all the processors
  debug(s"Start acceptor thread on listener ${endpoint.listenerName}")
  if (!acceptor.isStarted()) {
    KafkaThread.nonDaemon(
      s"${threadPrefix}-kafka-socket-acceptor-${endpoint.listenerName}-${endpoint.securityProtocol}-${endpoint.port}",
      acceptor
    ).start()// start the acceptor
    acceptor.awaitStartup()
  }
  info(s"Started $threadPrefix acceptor and processor(s) for endpoint : ${endpoint.listenerName}")
}
```

acceptor的startProcessors()

```scala
private[network] def startProcessors(processorThreadPrefix: String): Unit = synchronized {
  if (!processorsStarted.getAndSet(true)) {
    startProcessors(processors, processorThreadPrefix)
  }
}

private def startProcessors(processors: Seq[Processor], processorThreadPrefix: String): Unit = synchronized {
  processors.foreach { processor =>
    KafkaThread.nonDaemon(
      s"${processorThreadPrefix}-kafka-network-thread-$nodeId-${endPoint.listenerName}-${endPoint.securityProtocol}-${processor.id}",
      processor
    ).start()// 输出发现他调用了3个processor
  }
}
```



