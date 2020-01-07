---
date: 2019-09-18
title: "kafka-socketserver"
author: "邓子明"
tags:
    - kafka
    - 源码
categories:
    - kafka源码
comment: true
---


## SocketServer

注释可以看出来它的模型：
```
一个 Acceptor 用来接收新连接
每个 Acceptor 有 N 个 Processor 线程
每个 Processor 有一个 selector 用来读网络请求
还有 M 个 Handler 线程用来处理 request 并给 Processor 产生响应
```


核心的 field 介绍

```

// 一个 endpoint 数组，一般每个服务器有多个网卡，可以有多个 acceptor
private val endpoints = config.listeners 

// 每个 Acceptor 的线程数
private val numProcessorThreads = config.numNetworkThreads 
// 请求队列长度
private val maxQueuedRequests = config.queuedMaxRequests

// 总的 线程数
private val totalProcessorThreads = numProcessorThreads * endpoints.size

// 比较关键的一个类，控制请求的处理
val requestChannel = new RequestChannel(totalProcessorThreads, maxQueuedRequests)

// 所有的 Processor
private val processors = new Array[Processor](totalProcessorThreads)

// acceptor
private[network] val acceptors = mutable.Map[EndPoint, Acceptor]()

```

startup 方法

```
def startup() {
  this.synchronized {
	// 省略部分逻辑
    var processorBeginIndex = 0
    endpoints.values.foreach { endpoint =>
      val protocol = endpoint.protocolType
      val processorEndIndex = processorBeginIndex + numProcessorThreads

      for (i <- processorBeginIndex until processorEndIndex)
        // 新建 Processor
        processors(i) = newProcessor(i, connectionQuotas, protocol)

		// 新建 Acceptor
      val acceptor = new Acceptor(endpoint, sendBufferSize, recvBufferSize, brokerId,
        processors.slice(processorBeginIndex, processorEndIndex), connectionQuotas)
      
      // 开启线程
      acceptors.put(endpoint, acceptor)
      Utils.newThread("kafka-socket-acceptor-%s-%d".format(protocol.toString, endpoint.port), acceptor, false).start()
      acceptor.awaitStartup()

      processorBeginIndex = processorEndIndex
    }
  }
  
// newProcessor 方法 
protected[network] def newProcessor(id: Int, connectionQuotas: ConnectionQuotas, protocol: SecurityProtocol): Processor = {
  new Processor(id,
    time,
    config.socketRequestMaxBytes,
    requestChannel,
    connectionQuotas,
    config.connectionsMaxIdleMs,
    protocol,
    config.values,
    metrics
  )
}

// 方法就是直接新建一个 Processor，但是要注意几个参数，第一个 id 就是按顺序的编号，requestChannel 比较关键。
// 注意 Processor 和 Acceptor 都继承自 AbstractServerThread，里面有一些工具方法

```

然后我们可以看出最核心的就是 Processor 和 Acceptor 的 run 方法了，里面是最核心的逻辑。

```
 /**
   * Accept loop that checks for new connection attempts
   */
  def run() {
    serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
    startupComplete()
    try {
      var currentProcessor = 0
      while (isRunning) {
        try {
          val ready = nioSelector.select(500)
          if (ready > 0) {
            val keys = nioSelector.selectedKeys()
            val iter = keys.iterator()
            while (iter.hasNext && isRunning) {
              try {
                val key = iter.next
                iter.remove()
                if (key.isAcceptable)
                  accept(key, processors(currentProcessor))
                else
                  throw new IllegalStateException("Unrecognized key state for acceptor thread.")

                // round robin to the next processor thread
                currentProcessor = (currentProcessor + 1) % processors.length
              } catch {
                case e: Throwable => error("Error while accepting connection", e) 
                } } } }
        catch { // 省略 } } } finally { // 省略 }
  }

// 如果熟悉 Java nio 这段代码就没啥负责的，主要就是看 `accept(key, processors(currentProcessor))`

  /*
   * Accept a new connection
   */
  def accept(key: SelectionKey, processor: Processor) {
    val serverSocketChannel = key.channel().asInstanceOf[ServerSocketChannel]
    // 接受
    val socketChannel = serverSocketChannel.accept()
    try {
      connectionQuotas.inc(socketChannel.socket().getInetAddress)
      socketChannel.configureBlocking(false)
      socketChannel.socket().setTcpNoDelay(true)
      socketChannel.socket().setKeepAlive(true)
      socketChannel.socket().setSendBufferSize(sendBufferSize)

      debug("Accepted connection from %s on %s and assigned it to processor %d, sendBufferSize [actual|requested]: [%d|%d] recvBufferSize [actual|requested]: [%d|%d]"
            .format(socketChannel.socket.getRemoteSocketAddress, socketChannel.socket.getLocalSocketAddress, processor.id,
                  socketChannel.socket.getSendBufferSize, sendBufferSize,
                  socketChannel.socket.getReceiveBufferSize, recvBufferSize))
		
      processor.accept(socketChannel)
    } catch {
      case e: TooManyConnectionsException =>
        info("Rejected connection from %s, address already has the configured maximum of %d connections.".format(e.ip, e.count))
        close(socketChannel)
    }
  }
  
// 发现这段代码做的就是先 accept 然后调用 processor.accept(socketChannel)


  /**
   * Queue up a new connection for reading
   */
  def accept(socketChannel: SocketChannel) {
    newConnections.add(socketChannel)
    wakeup()
  }

// 而 Processor 的 Accept 方法很简单，直接添加到 newConnections 中
```

```
  override def run() {
    startupComplete()
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
      } catch {
        // ... 
        case e: ControlThrowable => throw e
        case e: Throwable =>
          error("Processor got uncaught exception.", e)
      }
    }

    debug("Closing selector - processor " + id)
    swallowError(closeAll())
    shutdownComplete()
  }
  
```

看到这个代码我们就会想起 NetworkClient 的代码，处理逻辑非常像。先处理要发送给客户端的响应，然后 poll，然后处理完成的接受和发送

configureNewConnections 用来注册所有的 Acceptor 接受的 SocketChannel

processNewResponses 从 requestChannel 取出当前 Processor 对应的 Response 调用 sendResponse。
requestChannel 里面的数据我们后续讨论，sendResponse 也不复杂


poll 方法就是调用 selector 的 poll

processCompletedReceives 就是调用 selector.completedReceives，然后遍历集合，将 Receive 放入 requestChannel 中

processCompletedSends 就是发送完成后的一些处理

我们看个细节，inflightResponses 集合和客户端的 inflightRequests 很像，我们看看他的调用点：
sendResponse 的时候调用 inflightResponses += (response.request.connectionId -> response)
processCompletedSends 的时候调用 inflightResponses.remove(send.destination)
processDisconnected 的时候调用 inflightResponses.remove(connectionId)

然后我们看看 selector 的管理 
processCompletedReceives 有一个 selector.mute(receive.source)
processNewResponses 的时候调用 selector.unmute(curr.request.connectionId)
processCompletedSends 的时候调用 selector.unmute(send.destination)

通过这个操作可以保证每次只接受一个请求，而且每次接受的请求必须处理完成才能接受下一个请求，保证请求和响应的有序性。


下面看一下 RequestChannel

上面有几个地方用到了 RequestChannel

SocketServer 中有 new RequestChannel(totalProcessorThreads, maxQueuedRequests)

newProcessor 的时候传入了上面 new 的 RequestChannel

processNewResponses 的时候 requestChannel.receiveResponse(id)， 

processCompletedReceives 的时候 requestChannel.sendRequest(req)


RequestChannel 的 属性

numProcessors 
queueSize
requestQueue 一个 ArrayBlockingQueue[RequestChannel.Request]，用于接收 request
responseQueues 一个 Array[BlockingQueue[RequestChannel.Response]] 用于返回 response

方法都比较简单。

上面我们看完了就想知道请求放进 channel 后谁来处理？我们只需要找一下方法调用就知道了，是 KafkaRequestHandlerPool 中处理的。



### 处理请求

KafkaRequestHandlerPool 内部实际上是有很多 KafkaRequestHandler，循环调用 requestChannel.receiveRequest(300)，然后调用 apis.handle(req)

这个 apis = new KafkaApis(socketServer.requestChannel, replicaManager, groupCoordinator, kafkaController, zkUtils, config.brokerId, config, metadataCache, metrics, authorizer)

apis 会根据请求种类处理请求，生成响应，然后调用 requestChannel.sendResponse(new Response(request, new ResponseSend(request.connectionId, respHeader, response)))

可以认为 卡夫卡 的绝大部分功能都可以查看 KafkaApis 进行分析。



