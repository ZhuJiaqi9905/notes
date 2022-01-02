# Server

最主体的处理部分：

- `core/src/main/scala/kafka/network/SocketServer.scala` 
- line 900

```scala
  override def run(): Unit = {
    startupComplete()
    try {
      while (isRunning) {
        try {
          // setup any new connections that have been queued up
          configureNewConnections()
          // register any new responses for writing
          processNewResponses()
          poll()
          processCompletedReceives()
          processCompletedSends()
          processDisconnected()
          closeExcessConnections()
        } catch {
          // We catch all the throwables here to prevent the processor thread from exiting. We do this because
          // letting a processor exit might cause a bigger impact on the broker. This behavior might need to be
          // reviewed if we see an exception that needs the entire broker to stop. Usually the exceptions thrown would
          // be either associated with a specific socket channel or a bad request. These exceptions are caught and
          // processed by the individual methods above which close the failing channel and continue processing other
          // channels. So this catch block should only ever see ControlThrowables.
          case e: Throwable => processException("Processor got uncaught exception.", e)
        }
      }
    } finally {
      debug(s"Closing selector - processor $id")
      CoreUtils.swallow(closeAll(), this, Level.ERROR)
      shutdownComplete()
    }
  }
```

其中最关键的就是中间那几步

```scala
  private def processNewResponses(): Unit = {
    var currentResponse: RequestChannel.Response = null
    while ({currentResponse = dequeueResponse(); currentResponse != null}) {
      val channelId = currentResponse.request.context.connectionId
      try {
        currentResponse match {
          case response: NoOpResponse =>
            // There is no response to send to the client, we need to read more pipelined requests
            // that are sitting in the server's socket buffer
            updateRequestMetrics(response)
            trace(s"Socket server received empty response to send, registering for read: $response")
            // Try unmuting the channel. If there was no quota violation and the channel has not been throttled,
            // it will be unmuted immediately. If the channel has been throttled, it will be unmuted only if the
            // throttling delay has already passed by now.
            handleChannelMuteEvent(channelId, ChannelMuteEvent.RESPONSE_SENT)
            tryUnmuteChannel(channelId)

          case response: SendResponse =>
            sendResponse(response, response.responseSend)
          case response: CloseConnectionResponse =>
            updateRequestMetrics(response)
            trace("Closing socket connection actively according to the response code.")
            close(channelId)
          case _: StartThrottlingResponse =>
            handleChannelMuteEvent(channelId, ChannelMuteEvent.THROTTLE_STARTED)
          case _: EndThrottlingResponse =>
            // Try unmuting the channel. The channel will be unmuted only if the response has already been sent out to
            // the client.
            handleChannelMuteEvent(channelId, ChannelMuteEvent.THROTTLE_ENDED)
            tryUnmuteChannel(channelId)
          case _ =>
            throw new IllegalArgumentException(s"Unknown response type: ${currentResponse.getClass}")
        }
      } catch {
        case e: Throwable =>
          processChannelException(channelId, s"Exception while processing response for $channelId", e)
      }
    }
  }
```



```scala
  private def poll(): Unit = {
    val pollTimeout = if (newConnections.isEmpty) 300 else 0
    try selector.poll(pollTimeout)
    catch {
      case e @ (_: IllegalStateException | _: IOException) =>
        // The exception is not re-thrown and any completed sends/receives/connections/disconnections
        // from this poll will be processed.
        error(s"Processor $id poll failed", e)
    }
  }
```

- 在newConnections时，pollTimeout的时间是0，意味着RdmaSelector只执行一次
- 

```scala
private def processCompletedReceives(): Unit = {
  selector.completedReceives.forEach { receive =>
    try {
      openOrClosingChannel(receive.source) match {
        case Some(channel) =>
          val header = parseRequestHeader(receive.payload)
          if (header.apiKey == ApiKeys.SASL_HANDSHAKE && channel.maybeBeginServerReauthentication(receive,
            () => time.nanoseconds()))
            trace(s"Begin re-authentication: $channel")
          else {
            val nowNanos = time.nanoseconds()
            if (channel.serverAuthenticationSessionExpired(nowNanos)) {
              // be sure to decrease connection count and drop any in-flight responses
              debug(s"Disconnecting expired channel: $channel : $header")
              close(channel.id)
              expiredConnectionsKilledCount.record(null, 1, 0)
            } else {
              val connectionId = receive.source
              val context = new RequestContext(header, connectionId, channel.socketAddress,
                channel.principal, listenerName, securityProtocol,
                channel.channelMetadataRegistry.clientInformation, isPrivilegedListener, channel.principalSerde)

              val req = new RequestChannel.Request(processor = id, context = context,
                startTimeNanos = nowNanos, memoryPool, receive.payload, requestChannel.metrics, None)

              // KIP-511: ApiVersionsRequest is intercepted here to catch the client software name
              // and version. It is done here to avoid wiring things up to the api layer.
              if (header.apiKey == ApiKeys.API_VERSIONS) {
                val apiVersionsRequest = req.body[ApiVersionsRequest]
                if (apiVersionsRequest.isValid) {
                  channel.channelMetadataRegistry.registerClientInformation(new ClientInformation(
                    apiVersionsRequest.data.clientSoftwareName,
                    apiVersionsRequest.data.clientSoftwareVersion))
                }
              }
              requestChannel.sendRequest(req)
              selector.mute(connectionId)
              handleChannelMuteEvent(connectionId, ChannelMuteEvent.REQUEST_RECEIVED)
            }
          }
        case None =>
          // This should never happen since completed receives are processed immediately after `poll()`
          throw new IllegalStateException(s"Channel ${receive.source} removed from selector before processing completed receive")
      }
    } catch {
      // note that even though we got an exception, we can assume that receive.source is valid.
      // Issues with constructing a valid receive object were handled earlier
      case e: Throwable =>
        processChannelException(receive.source, s"Exception while processing request from ${receive.source}", e)
    }
  }
  selector.clearCompletedReceives()
}

```





```scala
private def processCompletedSends(): Unit = {
  selector.completedSends.forEach { send =>
    try {
      val response = inflightResponses.remove(send.destinationId).getOrElse {
        throw new IllegalStateException(s"Send for ${send.destinationId} completed, but not in `inflightResponses`")
      }
      updateRequestMetrics(response)

      // Invoke send completion callback
      response.onComplete.foreach(onComplete => onComplete(send))

      // Try unmuting the channel. If there was no quota violation and the channel has not been throttled,
      // it will be unmuted immediately. If the channel has been throttled, it will unmuted only if the throttling
      // delay has already passed by now.
      handleChannelMuteEvent(send.destinationId, ChannelMuteEvent.RESPONSE_SENT)
      tryUnmuteChannel(send.destinationId)
    } catch {
      case e: Throwable => processChannelException(send.destinationId,
        s"Exception while processing completed send to ${send.destinationId}", e)
    }
  }
  selector.clearCompletedSends()
}
```





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



