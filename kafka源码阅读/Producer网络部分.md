# Producer网络部分

网络传输的最重要的两部分：

- 数据在哪
- 怎么发送

## KafkaProducer

Producer的实现类是`KafkaProducer`

```java
public class KafkaProducer<K, V> implements Producer<K, V> {
	private final Sender sender;
	private final RecordAccumulator accumulator;
	...
};
```

它只负责向Broker发送message，所以逻辑比较简单。其有关的方法是:

```java
/**
 * Asynchronously send a record to a topic. Equivalent to <code>send(record, null)</code>.
 * See {@link #send(ProducerRecord, Callback)} for details.
 */
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
    return send(record, null);
}
```

```java
/**
 * Asynchronously send a record to a topic and invoke the provided callback when the send has been acknowledged.
 * <p>
 * The send is asynchronous and this method will return immediately once the record has been stored in the buffer of
 * records waiting to be sent. This allows sending many records in parallel without blocking to wait for the
 * response after each one.
 * <p>
 * The result of the send is a {@link RecordMetadata} specifying the partition the record was sent to, the offset
 * it was assigned and the timestamp of the record. If
 * {@link org.apache.kafka.common.record.TimestampType#CREATE_TIME CreateTime} is used by the topic, the timestamp
 * will be the user provided timestamp or the record send time if the user did not specify a timestamp for the
 * record. If {@link org.apache.kafka.common.record.TimestampType#LOG_APPEND_TIME LogAppendTime} is used for the
 * topic, the timestamp will be the Kafka broker local time when the message is appended.
 * <p>
 * Since the send call is asynchronous it returns a {@link java.util.concurrent.Future Future} for the
 * {@link RecordMetadata} that will be assigned to this record. Invoking {@link java.util.concurrent.Future#get()
 * get()} on this future will block until the associated request completes and then return the metadata for the record
 * or throw any exception that occurred while sending the record.
 * <p>
 * If you want to simulate a simple blocking call you can call the <code>get()</code> method immediately:
 *
 * <pre>
 * {@code
 * byte[] key = "key".getBytes();
 * byte[] value = "value".getBytes();
 * ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("my-topic", key, value)
 * producer.send(record).get();
 * }</pre>
 * <p>
 * Fully non-blocking usage can make use of the {@link Callback} parameter to provide a callback that
 * will be invoked when the request is complete.
 *
 * <pre>
 * {@code
 * ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("the-topic", key, value);
 * producer.send(myRecord,
 *               new Callback() {
 *                   public void onCompletion(RecordMetadata metadata, Exception e) {
 *                       if(e != null) {
 *                          e.printStackTrace();
 *                       } else {
 *                          System.out.println("The offset of the record we just sent is: " + metadata.offset());
 *                       }
 *                   }
 *               });
 * }
 * </pre>
 *
 * Callbacks for records being sent to the same partition are guaranteed to execute in order. That is, in the
 * following example <code>callback1</code> is guaranteed to execute before <code>callback2</code>:
 *
 * <pre>
 * {@code
 * producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key1, value1), callback1);
 * producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key2, value2), callback2);
 * }
 * </pre>
 * <p>
 * When used as part of a transaction, it is not necessary to define a callback or check the result of the future
 * in order to detect errors from <code>send</code>. If any of the send calls failed with an irrecoverable error,
 * the final {@link #commitTransaction()} call will fail and throw the exception from the last failed send. When
 * this happens, your application should call {@link #abortTransaction()} to reset the state and continue to send
 * data.
 * </p>
 * <p>
 * Some transactional send errors cannot be resolved with a call to {@link #abortTransaction()}.  In particular,
 * if a transactional send finishes with a {@link ProducerFencedException}, a {@link org.apache.kafka.common.errors.OutOfOrderSequenceException},
 * a {@link org.apache.kafka.common.errors.UnsupportedVersionException}, or an
 * {@link org.apache.kafka.common.errors.AuthorizationException}, then the only option left is to call {@link #close()}.
 * Fatal errors cause the producer to enter a defunct state in which future API calls will continue to raise
 * the same underyling error wrapped in a new {@link KafkaException}.
 * </p>
 * <p>
 * It is a similar picture when idempotence is enabled, but no <code>transactional.id</code> has been configured.
 * In this case, {@link org.apache.kafka.common.errors.UnsupportedVersionException} and
 * {@link org.apache.kafka.common.errors.AuthorizationException} are considered fatal errors. However,
 * {@link ProducerFencedException} does not need to be handled. Additionally, it is possible to continue
 * sending after receiving an {@link org.apache.kafka.common.errors.OutOfOrderSequenceException}, but doing so
 * can result in out of order delivery of pending messages. To ensure proper ordering, you should close the
 * producer and create a new instance.
 * </p>
 * <p>
 * If the message format of the destination topic is not upgraded to 0.11.0.0, idempotent and transactional
 * produce requests will fail with an {@link org.apache.kafka.common.errors.UnsupportedForMessageFormatException}
 * error. If this is encountered during a transaction, it is possible to abort and continue. But note that future
 * sends to the same topic will continue receiving the same exception until the topic is upgraded.
 * </p>
 * <p>
 * Note that callbacks will generally execute in the I/O thread of the producer and so should be reasonably fast or
 * they will delay the sending of messages from other threads. If you want to execute blocking or computationally
 * expensive callbacks it is recommended to use your own {@link java.util.concurrent.Executor} in the callback body
 * to parallelize processing.
 *
 * @param record The record to send
 * @param callback A user-supplied callback to execute when the record has been acknowledged by the server (null
 *        indicates no callback)
 *
 * @throws AuthenticationException if authentication fails. See the exception for more details
 * @throws AuthorizationException fatal error indicating that the producer is not allowed to write
 * @throws IllegalStateException if a transactional.id has been configured and no transaction has been started, or
 *                               when send is invoked after producer has been closed.
 * @throws InterruptException If the thread is interrupted while blocked
 * @throws SerializationException If the key or value are not valid objects given the configured serializers
 * @throws KafkaException If a Kafka related error occurs that does not belong to the public API exceptions.
 */
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}
```

可以看到，`send()`方法是调用了`doSend()`方法作为具体的发送逻辑，因此我们来看`doSend()`方法：

```java
/**
 * Implementation of asynchronously send a record to a topic.
 */
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        throwIfProducerClosed();
        // first make sure the metadata for the topic is available
        long nowMs = time.milliseconds();
        ClusterAndWaitTime clusterAndWaitTime;
        try {
            clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
        } catch (KafkaException e) {
            if (metadata.isClosed())
                throw new KafkaException("Producer closed while send in progress", e);
            throw e;
        }
        nowMs += clusterAndWaitTime.waitedOnMetadataMs;
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);// 更新能等待的时间
        Cluster cluster = clusterAndWaitTime.cluster;
        byte[] serializedKey;
        try {
            serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());// 序列化
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in key.serializer", cce);
        }
        byte[] serializedValue;
        try {
            serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in value.serializer", cce);
        }
        int partition = partition(record, serializedKey, serializedValue, cluster);
        tp = new TopicPartition(record.topic(), partition);// partition

        setReadOnly(record.headers());
        Header[] headers = record.headers().toArray();

        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                compressionType, serializedKey, serializedValue, headers);// 估计能放下一个record的batch size的上界
        ensureValidRecordSize(serializedSize);// 确保序列化的长度不太大
        long timestamp = record.timestamp() == null ? nowMs : record.timestamp();
        if (log.isTraceEnabled()) {
            log.trace("Attempting to append record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
        }
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);// 建立拦截器的callback函数

        if (transactionManager != null && transactionManager.isTransactional()) {
            transactionManager.failIfNotReadyForSend();
        }
        // 添加要发送的消息到accumulator
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);

        if (result.abortForNewBatch) {// 丢弃原来的batch,放到一个新的batch中？
            int prevPartition = partition;
            partitioner.onNewBatch(record.topic(), cluster, prevPartition);
            partition = partition(record, serializedKey, serializedValue, cluster);
            tp = new TopicPartition(record.topic(), partition);
            if (log.isTraceEnabled()) {
                log.trace("Retrying append due to new batch creation for topic {} partition {}. The old partition was {}", record.topic(), partition, prevPartition);
            }
            // producer callback will make sure to call both 'callback' and interceptor callback
            interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

            result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs, false, nowMs);
        }

        if (transactionManager != null && transactionManager.isTransactional())
            transactionManager.maybeAddPartitionToTransaction(tp);

        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();// 唤醒sender
        }
        return result.future;
        // handling exceptions and record the errors;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
    } catch (ApiException e) {
        log.debug("Exception occurred during message send:", e);
        if (callback != null)
            callback.onCompletion(null, e);
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        return new FutureFailure(e);
    } catch (InterruptedException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw new InterruptException(e);
    } catch (KafkaException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw e;
    } catch (Exception e) {
        // we notify interceptor about all exceptions, since onSend is called before anything else in this method
        this.interceptors.onSendError(record, tp, e);
        throw e;
    }
}
```

流程：

- 等待cluster中关于这个topic和partition的metadata可用:`clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);`
- 更新”能等待的时间“
- 把要发送的数据（key, value）进行序列化
- 按照record进行分区：`int partition = partition(record, serializedKey, serializedValue, cluster);`
- 建立拦截器的callback函数
-  添加（缓存）要发送的消息到accumulator
-  唤醒sender

和网络传输有关的是sender和accumulator，accumulator中存放要发送的数据，sender是将其中的数据发送出去。

那么我们先来看一下sender, 它是`Sender`类:

## Sender

```java
public class Sender implements Runnable{
	/* the state of each nodes connection */
  private final KafkaClient client;
  
  /* the record accumulator that batches records */
  private final RecordAccumulator accumulator;
   
  // A per-partition queue of batches ordered by creation time for tracking the in-flight batches
  private final Map<TopicPartition, List<ProducerBatch>> inFlightBatches;
  
  ...
}
```

注意到Sender类持有了RecordAccumulator的引用，因此能得到要发送的数据。

首先wakeup()方法唤醒了Sender的client

```java
  /**
   * Wake up the selector associated with this send thread
   */
  public void wakeup() {
      this.client.wakeup();
  }
```

而Sender的client是一个KafkaClient接口;我们之后会看一下KafkaClient,但是这里我们先暂停一下。

因为这个Sender实现了Runnable接口，说明它很有可能是在一个独立的线程中运行。KafkaProducer的构造函数也证明了这一点：

```java
// 构造函数中的一部分
this.sender = newSender(logContext, kafkaClient, this.metadata);
String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;
this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
this.ioThread.start();
```

所以说Sender的run()方法是需要详细看的部分：

我略去了在关闭客户端时的错误处理部分，他的主要部分如下

```java
@Override
public void run() {
    log.debug("Starting Kafka producer I/O thread.");

    // main loop, runs until close is called
    while (running) {
        try {
            runOnce();// 主要内容
        } catch (Exception e) {
            log.error("Uncaught error in kafka producer I/O thread: ", e);
        }
    }

    log.debug("Beginning shutdown of Kafka producer I/O thread, sending remaining records.");


}
```

那么主要内容就是runOnce():

```java
void runOnce() {
    if (transactionManager != null) {
        try {
            transactionManager.maybeResolveSequences();

            // do not continue sending if the transaction manager is in a failed state
            if (transactionManager.hasFatalError()) {
                RuntimeException lastError = transactionManager.lastError();
                if (lastError != null)
                    maybeAbortBatches(lastError);
                client.poll(retryBackoffMs, time.milliseconds());
                return;
            }

            // Check whether we need a new producerId. If so, we will enqueue an InitProducerId
            // request which will be sent below
            transactionManager.bumpIdempotentEpochAndResetIdIfNeeded();

            if (maybeSendAndPollTransactionalRequest()) {
                return;
            }
        } catch (AuthenticationException e) {
            // This is already logged as error, but propagated here to perform any clean ups.
            log.trace("Authentication exception while processing transactional request", e);
            transactionManager.authenticationFailed(e);
        }
    }

    long currentTimeMs = time.milliseconds();
  // 发送数据的部分
    long pollTimeout = sendProducerData(currentTimeMs);
    client.poll(pollTimeout, currentTimeMs);
}
```

发现最后两行代码是主要进行发送数据的部分，所以我们认真看一下：

```java
private long sendProducerData(long now) {
    Cluster cluster = metadata.fetch();
    // get the list of partitions with data ready to send
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

    // if there are any partitions whose leaders are not known yet, force metadata update
    if (!result.unknownLeaderTopics.isEmpty()) {
        // The set of topics with unknown leader contains topics with leader election pending as well as
        // topics which may have expired. Add the topic again to metadata to ensure it is included
        // and request metadata update, since there are messages to send to the topic.
        for (String topic : result.unknownLeaderTopics)
            this.metadata.add(topic, now);

        log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}",
            result.unknownLeaderTopics);
        this.metadata.requestUpdate();
    }

    // remove any nodes we aren't ready to send to
    Iterator<Node> iter = result.readyNodes.iterator();
    long notReadyTimeout = Long.MAX_VALUE;
    while (iter.hasNext()) {
        Node node = iter.next();
        if (!this.client.ready(node, now)) {
            iter.remove();
            notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
        }
    }

    // create produce requests
    Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);
    addToInflightBatches(batches);
    if (guaranteeMessageOrder) {
        // Mute all the partitions drained
        for (List<ProducerBatch> batchList : batches.values()) {
            for (ProducerBatch batch : batchList)
                this.accumulator.mutePartition(batch.topicPartition);
        }
    }

    accumulator.resetNextBatchExpiryTime();
    List<ProducerBatch> expiredInflightBatches = getExpiredInflightBatches(now);
    List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(now);
    expiredBatches.addAll(expiredInflightBatches);

    // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics
    // for expired batches. see the documentation of @TransactionState.resetIdempotentProducerId to understand why
    // we need to reset the producer id here.
    if (!expiredBatches.isEmpty())
        log.trace("Expired {} batches in accumulator", expiredBatches.size());
    for (ProducerBatch expiredBatch : expiredBatches) {
        String errorMessage = "Expiring " + expiredBatch.recordCount + " record(s) for " + expiredBatch.topicPartition
            + ":" + (now - expiredBatch.createdMs) + " ms has passed since batch creation";
        failBatch(expiredBatch, -1, NO_TIMESTAMP, new TimeoutException(errorMessage), false);
        if (transactionManager != null && expiredBatch.inRetry()) {
            // This ensures that no new batches are drained until the current in flight batches are fully resolved.
            transactionManager.markSequenceUnresolved(expiredBatch);
        }
    }
    sensors.updateProduceRequestMetrics(batches);

    // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
    // loop and try sending more data. Otherwise, the timeout will be the smaller value between next batch expiry
    // time, and the delay time for checking data availability. Note that the nodes may have data that isn't yet
    // sendable due to lingering, backing off, etc. This specifically does not include nodes with sendable data
    // that aren't ready to send since they would cause busy looping.
    long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
    pollTimeout = Math.min(pollTimeout, this.accumulator.nextExpiryTimeMs() - now);
    pollTimeout = Math.max(pollTimeout, 0);
    if (!result.readyNodes.isEmpty()) {
        log.trace("Nodes with data ready to send: {}", result.readyNodes);
        // if some partitions are already ready to be sent, the select time would be 0;
        // otherwise if some partition already has some data accumulated but not ready yet,
        // the select time will be the time difference between now and its linger expiry time;
        // otherwise the select time will be the time difference between now and the metadata expiry time;
        pollTimeout = 0;
    }
    sendProduceRequests(batches, now);//
    return pollTimeout;
}
```

sendProducerData()函数首先把原本的`<Partition, Deque<ProducerBatch>>`的形式转化为`<Integer, List<ProducerBatch>>`的形式，然后调用sendProduceRequests()函数发送数据。

```java
/**
 * Transfer the record batches into a list of produce requests on a per-node basis
 */
private void sendProduceRequests(Map<Integer, List<ProducerBatch>> collated, long now) {
    for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet())
        sendProduceRequest(now, entry.getKey(), acks, requestTimeoutMs, entry.getValue());
}
```

sendProduceRequests()函数会对于每个Node来调用sendProduceRequest()函数发送数据。

下面来看一下sendProductRequest()函数

```java
private void sendProduceRequest(long now, int destination, short acks, int timeout, List<ProducerBatch> batches) {
    if (batches.isEmpty())
        return;

    final Map<TopicPartition, ProducerBatch> recordsByPartition = new HashMap<>(batches.size());

    // find the minimum magic version used when creating the record sets
    byte minUsedMagic = apiVersions.maxUsableProduceMagic();
    for (ProducerBatch batch : batches) {
        if (batch.magic() < minUsedMagic)
            minUsedMagic = batch.magic();
    }
    ProduceRequestData.TopicProduceDataCollection tpd = new ProduceRequestData.TopicProduceDataCollection();
    for (ProducerBatch batch : batches) {
        TopicPartition tp = batch.topicPartition;
        MemoryRecords records = batch.records();

        // down convert if necessary to the minimum magic used. In general, there can be a delay between the time
        // that the producer starts building the batch and the time that we send the request, and we may have
        // chosen the message format based on out-dated metadata. In the worst case, we optimistically chose to use
        // the new message format, but found that the broker didn't support it, so we need to down-convert on the
        // client before sending. This is intended to handle edge cases around cluster upgrades where brokers may
        // not all support the same message format version. For example, if a partition migrates from a broker
        // which is supporting the new magic version to one which doesn't, then we will need to convert.
        if (!records.hasMatchingMagic(minUsedMagic))
            records = batch.records().downConvert(minUsedMagic, 0, time).records();
        ProduceRequestData.TopicProduceData tpData = tpd.find(tp.topic());
        if (tpData == null) {
            tpData = new ProduceRequestData.TopicProduceData().setName(tp.topic());
            tpd.add(tpData);
        }
        tpData.partitionData().add(new ProduceRequestData.PartitionProduceData()
                .setIndex(tp.partition())
                .setRecords(records));
        recordsByPartition.put(tp, batch);
    }

    String transactionalId = null;
    if (transactionManager != null && transactionManager.isTransactional()) {
        transactionalId = transactionManager.transactionalId();
    }

    ProduceRequest.Builder requestBuilder = ProduceRequest.forMagic(minUsedMagic,
            new ProduceRequestData()
                    .setAcks(acks)
                    .setTimeoutMs(timeout)
                    .setTransactionalId(transactionalId)
                    .setTopicData(tpd));
    RequestCompletionHandler callback = response -> handleProduceResponse(response, recordsByPartition, time.milliseconds());

    String nodeId = Integer.toString(destination);
    ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,
            requestTimeoutMs, callback);
    client.send(clientRequest, now);// 这里发送了数据
    log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);
}
```

结论：

- 数据在哪：`clientRequest`
- 怎么发送：数据通过`client.send()`发送

**这里面有一个离谱的事情：找不到ProduceRequestData类。反正IDEA是在飘红的，我也没有搜到**

## ClientRequest

```java
public final class ClientRequest {

  private final String destination;
  private final AbstractRequest.Builder<?> requestBuilder;
  private final int correlationId;
  private final String clientId;
  private final long createdTimeMs;
  private final boolean expectResponse;
  private final int requestTimeoutMs;
  private final RequestCompletionHandler callback;
}
```

可以看到数据也不是存在ClientRequest中，而是它持有了一个requestBuider。

## AbstractRequest.Builder

```java
public abstract class AbstractRequest implements AbstractRequestResponse {

	public static abstract class Builder<T extends AbstractRequest> {
    public final Send toSend(RequestHeader header) {
          return SendBuilder.buildRequestSend(header, data());
      }
    ...
  };
};
```

这个`toSend()`方法里面构建了Request,并以Send接口封装返回。

## Send

是一个接口

- 这是一个很可怕的事情，因为这意味着底层的数据不是在Send里面，而可能是Send持有它的引用

```java
public interface Send {

  /**
   * Is this send complete?
   */
  boolean completed();

  /**
   * Write some as-yet unwritten bytes from this send to the provided channel. It may take multiple calls for the send
   * to be completely written
   * @param channel The Channel to write to
   * @return The number of bytes written
   * @throws IOException If the write fails
   */
  long writeTo(TransferableChannel channel) throws IOException;

  /**
   * Size of the send
   */
  long size();

}
```



## KafkaClient

```java
public interface KafkaClient extends Closeable {
  /**
   * Queue up the given request for sending. Requests can only be sent on ready connections.
   * @param request The request
   * @param now The current timestamp
   */
	void send(ClientRequest request, long now);
	/**
   * Do actual reads and writes from sockets.
   *
   * @param timeout The maximum amount of time to wait for responses in ms, must be non-negative. The implementation is free to use a lower value if appropriate (common reasons for this are a lower request or metadata update timeout)
   * @param now The current time in ms
   * @throws IllegalStateException If a request is sent to an unready node
   */
	List<ClientResponse> poll(long timeout, long now);
  /**
   * Wake up the client if it is currently blocked waiting for I/O
   */
  void wakeup();
  
  ...
};
  
```

这是一个接口，尴尬的是你不知道它的具体实现是什么，所以我们还要回到KafkaProducer中看一下在new 一个Sender时，传入的KafkaClient的具体实现。

在KafkaProducer的`newSender()`方法中，我们看到了：

```java
KafkaClient client = kafkaClient != null ? kafkaClient : new NetworkClient(
                new Selector(producerConfig.getLong(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG),
                        this.metrics, time, "producer", channelBuilder, logContext),
                metadata,
                clientId,
                maxInflightRequests,
                producerConfig.getLong(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                producerConfig.getLong(ProducerConfig.RECONNECT_BACKOFF_MAX_MS_CONFIG),
                producerConfig.getInt(ProducerConfig.SEND_BUFFER_CONFIG),
                producerConfig.getInt(ProducerConfig.RECEIVE_BUFFER_CONFIG),
                requestTimeoutMs,
                producerConfig.getLong(ProducerConfig.SOCKET_CONNECTION_SETUP_TIMEOUT_MS_CONFIG),
                producerConfig.getLong(ProducerConfig.SOCKET_CONNECTION_SETUP_TIMEOUT_MAX_MS_CONFIG),
                ClientDnsLookup.forConfig(producerConfig.getString(ProducerConfig.CLIENT_DNS_LOOKUP_CONFIG)),
                time,
                true,
                apiVersions,
                throttleTimeSensor,
                logContext);
```

所以说KafkaClient的一个具体实现是NetworkClient。

## NetworkClient

好了我们来看看这个类，要注意比较重要的方法：`send()`, `poll()`, `wakeup()`

```java
public class NetworkClient implements KafkaClient {

    private enum State {
        ACTIVE,
        CLOSING,
        CLOSED
    }
    /* the selector used to perform network i/o */
    private final Selectable selector;

    /* the socket send buffer size in bytes */
    private final int socketSendBuffer;

    /* the socket receive size buffer in bytes */
    private final int socketReceiveBuffer;

    /* the client id used to identify this client in requests to the server */
    private final String clientId;

  ...
};
```

注意到建立连接到函数里面，把sockerSendBuffer和socketReceiveBuffer注册到selector里面：

```java
private void initiateConnect(Node node, long now) {
    String nodeConnectionId = node.idString();
    try {
        connectionStates.connecting(nodeConnectionId, now, node.host(), clientDnsLookup);
        InetAddress address = connectionStates.currentAddress(nodeConnectionId);
        log.debug("Initiating connection to node {} using address {}", node, address);
        selector.connect(nodeConnectionId,
                new InetSocketAddress(address, node.port()),
                this.socketSendBuffer,
                this.socketReceiveBuffer);
    } catch (IOException e) {
        log.warn("Error connecting to node {}", node, e);
        // Attempt failed, we'll try again after the backoff
        connectionStates.disconnected(nodeConnectionId, now);
        // Notify metadata updater of the connection failure
        metadataUpdater.handleServerDisconnect(now, nodeConnectionId, Optional.empty());
    }
}
```

poll函数：

```java
/**
 * Do actual reads and writes to sockets.
 *
 * @param timeout The maximum amount of time to wait (in ms) for responses if there are none immediately,
 *                must be non-negative. The actual timeout will be the minimum of timeout, request timeout and
 *                metadata timeout
 * @param now The current time in milliseconds
 * @return The list of responses received
 */
@Override
public List<ClientResponse> poll(long timeout, long now) {
    ensureActive();

    if (!abortedSends.isEmpty()) {
        // If there are aborted sends because of unsupported version exceptions or disconnects,
        // handle them immediately without waiting for Selector#poll.
        List<ClientResponse> responses = new ArrayList<>();
        handleAbortedSends(responses);
        completeResponses(responses);
        return responses;
    }

    long metadataTimeout = metadataUpdater.maybeUpdate(now);
    try {
        this.selector.poll(Utils.min(timeout, metadataTimeout, defaultRequestTimeoutMs));// 调用了selector的poll
    } catch (IOException e) {
        log.error("Unexpected error during I/O", e);
    }

    // process completed actions
    long updatedNow = this.time.milliseconds();
    List<ClientResponse> responses = new ArrayList<>();
    handleCompletedSends(responses, updatedNow);
    handleCompletedReceives(responses, updatedNow);
    handleDisconnections(responses, updatedNow);
    handleConnections();
    handleInitiateApiVersionRequests(updatedNow);
    handleTimedOutConnections(responses, updatedNow);
    handleTimedOutRequests(responses, updatedNow);
    completeResponses(responses);

    return responses;
}
```

主要是调用了selector.poll()函数。然后是处理response。

```java
/**
 * Handle any completed request send. In particular if no response is expected consider the request complete.
 *
 * @param responses The list of responses to update
 * @param now The current time
 */
private void handleCompletedSends(List<ClientResponse> responses, long now) {
    // if no response is expected then when the send is completed, return it
    for (NetworkSend send : this.selector.completedSends()) {
        InFlightRequest request = this.inFlightRequests.lastSent(send.destinationId());
        if (!request.expectResponse) {
            this.inFlightRequests.completeLastSent(send.destinationId());
            responses.add(request.completed(null, now));
        }
    }
}
```

```java
/**
 * Handle any completed receives and update the response list with the responses received.
 *
 * @param responses The list of responses to update
 * @param now The current time
 */
private void handleCompletedReceives(List<ClientResponse> responses, long now) {
    for (NetworkReceive receive : this.selector.completedReceives()) {
        String source = receive.source();
        InFlightRequest req = inFlightRequests.completeNext(source);

        AbstractResponse response = parseResponse(receive.payload(), req.header);
        if (throttleTimeSensor != null)
            throttleTimeSensor.record(response.throttleTimeMs(), now);

        if (log.isDebugEnabled()) {
            log.debug("Received {} response from node {} for request with header {}: {}",
                req.header.apiKey(), req.destination, req.header, response);
        }

        // If the received response includes a throttle delay, throttle the connection.
        maybeThrottle(response, req.header.apiVersion(), req.destination, now);
        if (req.isInternalRequest && response instanceof MetadataResponse)
            metadataUpdater.handleSuccessfulResponse(req.header, now, (MetadataResponse) response);
        else if (req.isInternalRequest && response instanceof ApiVersionsResponse)
            handleApiVersionsResponse(responses, req, now, (ApiVersionsResponse) response);
        else
            responses.add(req.completed(response, now));
    }
}
```

还有其他的handle函数。

send函数：

```java
/**
 * Queue up the given request for sending. Requests can only be sent out to ready nodes.
 * @param request The request
 * @param now The current timestamp
 */
@Override
public void send(ClientRequest request, long now) {
    doSend(request, false, now);
}
```

调用了下面的doSend()

```java
private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now) {
    ensureActive();
    String nodeId = clientRequest.destination();
    if (!isInternalRequest) {
        // If this request came from outside the NetworkClient, validate
        // that we can send data.  If the request is internal, we trust
        // that internal code has done this validation.  Validation
        // will be slightly different for some internal requests (for
        // example, ApiVersionsRequests can be sent prior to being in
        // READY state.)
        if (!canSendRequest(nodeId, now))
            throw new IllegalStateException("Attempt to send a request to node " + nodeId + " which is not ready.");
    }
    AbstractRequest.Builder<?> builder = clientRequest.requestBuilder();
    try {
        NodeApiVersions versionInfo = apiVersions.get(nodeId);
        short version;
        // Note: if versionInfo is null, we have no server version information. This would be
        // the case when sending the initial ApiVersionRequest which fetches the version
        // information itself.  It is also the case when discoverBrokerVersions is set to false.
        if (versionInfo == null) {
            version = builder.latestAllowedVersion();
            if (discoverBrokerVersions && log.isTraceEnabled())
                log.trace("No version information found when sending {} with correlation id {} to node {}. " +
                        "Assuming version {}.", clientRequest.apiKey(), clientRequest.correlationId(), nodeId, version);
        } else {
            version = versionInfo.latestUsableVersion(clientRequest.apiKey(), builder.oldestAllowedVersion(),
                    builder.latestAllowedVersion());
        }
        // The call to build may also throw UnsupportedVersionException, if there are essential
        // fields that cannot be represented in the chosen version.
      // 这里调用了doSend()发送数据
      //这个builder是ProduceRequest.Builder,它调用了自己的build方法，返回一个ProduceRequest对象
        doSend(clientRequest, isInternalRequest, now, builder.build(version));
    } catch (UnsupportedVersionException unsupportedVersionException) {
        // If the version is not supported, skip sending the request over the wire.
        // Instead, simply add it to the local queue of aborted requests.
        log.debug("Version mismatch when attempting to send {} with correlation id {} to {}", builder,
                clientRequest.correlationId(), clientRequest.destination(), unsupportedVersionException);
        ClientResponse clientResponse = new ClientResponse(clientRequest.makeHeader(builder.latestAllowedVersion()),
                clientRequest.callback(), clientRequest.destination(), now, now,
                false, unsupportedVersionException, null, null);

        if (!isInternalRequest)
            abortedSends.add(clientResponse);
        else if (clientRequest.apiKey() == ApiKeys.METADATA)
            metadataUpdater.handleFailedRequest(now, Optional.of(unsupportedVersionException));
    }
}
```

它调用了下面的doSend()

```java
    private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now, AbstractRequest request) {
    String destination = clientRequest.destination();
    RequestHeader header = clientRequest.makeHeader(request.version());
    if (log.isDebugEnabled()) {
        log.debug("Sending {} request with header {} and timeout {} to node {}: {}",
            clientRequest.apiKey(), header, clientRequest.requestTimeoutMs(), destination, request);
    }
    Send send = request.toSend(header);// 这个toSend调用了AbstractRequest.toSend()方法
    InFlightRequest inFlightRequest = new InFlightRequest(
            clientRequest,
            header,
            isInternalRequest,
            request,
            send,
            now);
    this.inFlightRequests.add(inFlightRequest);
    selector.send(new NetworkSend(clientRequest.destination(), send));
}
```

看到这里是调用了selector的send方法。这个selector是`Selector`类。

我们先暂停一下，看看数据在哪？所以要深入理解一下`toSend()`方法。哦这还是个static方法。

```java
public abstract class AbstractRequest implements AbstractRequestResponse {

  public static abstract class Builder<T extends AbstractRequest> {

    public final Send toSend(RequestHeader header) {
        return SendBuilder.buildRequestSend(header, data());
    }
    ...
 };
  ...
};
```

好的我们需要看看SendBuilder：

## SendBuilder

```java

public class SendBuilder implements Writable {
  private final ByteBuffer buffer;

  private final Queue<Send> sends = new ArrayDeque<>(1);
  private long sizeOfSends = 0;

  private final List<ByteBuffer> buffers = new ArrayList<>();
  private long sizeOfBuffers = 0;
    
    
  public static Send buildRequestSend(
        RequestHeader header,
        Message apiRequest
    ) {
        return buildSend(
            header.data(),
            header.headerVersion(),
            apiRequest,
            header.apiVersion()
        );
    }


    private static Send buildSend(
        Message header,
        short headerVersion,
        Message apiMessage,
        short apiVersion
    ) {
        ObjectSerializationCache serializationCache = new ObjectSerializationCache();

        MessageSizeAccumulator messageSize = new MessageSizeAccumulator();
        header.addSize(messageSize, serializationCache, headerVersion);
        apiMessage.addSize(messageSize, serializationCache, apiVersion);
				// 这里new了一个builder，然后把数据写到builder的buffer里面了
        SendBuilder builder = new SendBuilder(messageSize.sizeExcludingZeroCopy() + 4);
        builder.writeInt(messageSize.totalSize());
        header.write(builder, serializationCache, headerVersion);
        apiMessage.write(builder, serializationCache, apiVersion);

        return builder.build();
    }

    public Send build() {
      flushPendingSend();
      if (sends.size() == 1) {
          return sends.poll();
      } else {
          return new MultiRecordsSend(sends, sizeOfSends);
      }
  	}
  
  private void flushPendingSend() {
        flushPendingBuffer();
        if (!buffers.isEmpty()) {
            ByteBuffer[] byteBufferArray = buffers.toArray(new ByteBuffer[0]);
            addSend(new ByteBufferSend(byteBufferArray, sizeOfBuffers));
            clearBuffers();
        }
    }
  
  private void addSend(Send send) {
        sends.add(send);
        sizeOfSends += send.size();
    }
  
  private void flushPendingBuffer() {
      int latestPosition = buffer.position();
      buffer.reset();

      if (latestPosition > buffer.position()) {
          buffer.limit(latestPosition);
          addBuffer(buffer.slice());

          buffer.position(latestPosition);
          buffer.limit(buffer.capacity());
          buffer.mark();
      }
  }
};
```

这个代码可太秀了。首先我们的static的`buildSend()`方法里面new了一个builder，然后把数据写到builder的buffer里面了。最后一行调用`builder.build()`方法。

sends是一个队列，每个元素都是Send接口，而它的实现是ByteBufferSend。

如果之后sends只有一个元素，就返回一个ByteBufferSend；如果有多个元素，就返回MultiRecordsSend。但是这两个类都实现了Send接口，所以`build()`方法的返回值是Send。

**说实话这个设计真的神了，但是这一层层的下去也太恶心了。**

所以我们再看看ByteBufferSend和MultiRecordsSend。

## ByteBufferSend

```java
/**
 * A send backed by an array of byte buffers
 */
public class ByteBufferSend implements Send {

    private final long size;
    protected final ByteBuffer[] buffers;
    private long remaining;
    private boolean pending = false;

    public ByteBufferSend(ByteBuffer... buffers) {
        this.buffers = buffers;
        for (ByteBuffer buffer : buffers)
            remaining += buffer.remaining();
        this.size = remaining;
    }

    public ByteBufferSend(ByteBuffer[] buffers, long size) {
        this.buffers = buffers;
        this.size = size;
        this.remaining = size;
    }

    @Override
    public boolean completed() {
        return remaining <= 0 && !pending;
    }

    @Override
    public long size() {
        return this.size;
    }

    @Override
    public long writeTo(TransferableChannel channel) throws IOException {
        long written = channel.write(buffers);
        if (written < 0)
            throw new EOFException("Wrote negative bytes to channel. This shouldn't happen.");
        remaining -= written;
        pending = channel.hasPendingWrites();
        return written;
    }

    public long remaining() {
        return remaining;
    }

    @Override
    public String toString() {
        return "ByteBufferSend(" +
            ", size=" + size +
            ", remaining=" + remaining +
            ", pending=" + pending +
            ')';
    }

    public static ByteBufferSend sizePrefixed(ByteBuffer buffer) {
        ByteBuffer sizeBuffer = ByteBuffer.allocate(4);
        sizeBuffer.putInt(0, buffer.remaining());
        return new ByteBufferSend(sizeBuffer, buffer);
    }
}

```

里面有一个ByteBuffer数组来保存数据，就是buffers。然后writeTo方法是调用channel的write来把buffers中的数据写入到channel。

## MultiRecordsSend

```java
**
 * A set of composite sends with nested {@link RecordsSend}, sent one after another
 */
public class MultiRecordsSend implements Send {
    private static final Logger log = LoggerFactory.getLogger(MultiRecordsSend.class);

    private final Queue<Send> sendQueue;
    private final long size;
    private Map<TopicPartition, RecordConversionStats> recordConversionStats;

    private long totalWritten = 0;
    private Send current;

    /**
     * Construct a MultiRecordsSend from a queue of Send objects. The queue will be consumed as the MultiRecordsSend
     * progresses (on completion, it will be empty).
     */
    public MultiRecordsSend(Queue<Send> sends) {
        this.sendQueue = sends;

        long size = 0;
        for (Send send : sends)
            size += send.size();
        this.size = size;

        this.current = sendQueue.poll();
    }

    public MultiRecordsSend(Queue<Send> sends, long size) {
        this.sendQueue = sends;
        this.size = size;
        this.current = sendQueue.poll();
    }

    @Override
    public long size() {
        return size;
    }

    @Override
    public boolean completed() {
        return current == null;
    }

    // Visible for testing
    int numResidentSends() {
        int count = 0;
        if (current != null)
            count += 1;
        count += sendQueue.size();
        return count;
    }

    @Override
    public long writeTo(TransferableChannel channel) throws IOException {
        if (completed())
            throw new KafkaException("This operation cannot be invoked on a complete request.");

        int totalWrittenPerCall = 0;
        boolean sendComplete;
        do {
            long written = current.writeTo(channel);
            totalWrittenPerCall += written;
            sendComplete = current.completed();
            if (sendComplete) {
                updateRecordConversionStats(current);
                current = sendQueue.poll();
            }
        } while (!completed() && sendComplete);

        totalWritten += totalWrittenPerCall;

        if (completed() && totalWritten != size)
            log.error("mismatch in sending bytes over socket; expected: {} actual: {}", size, totalWritten);

        log.trace("Bytes written as part of multi-send call: {}, total bytes written so far: {}, expected bytes to write: {}",
                totalWrittenPerCall, totalWritten, size);

        return totalWrittenPerCall;
    }

    /**
     * Get any statistics that were recorded as part of executing this {@link MultiRecordsSend}.
     * @return Records processing statistics (could be null if no statistics were collected)
     */
    public Map<TopicPartition, RecordConversionStats> recordConversionStats() {
        return recordConversionStats;
    }

    @Override
    public String toString() {
        return "MultiRecordsSend(" +
            "size=" + size +
            ", totalWritten=" + totalWritten +
            ')';
    }

    private void updateRecordConversionStats(Send completedSend) {
        // The underlying send might have accumulated statistics that need to be recorded. For example,
        // LazyDownConversionRecordsSend accumulates statistics related to the number of bytes down-converted, the amount
        // of temporary memory used for down-conversion, etc. Pull out any such statistics from the underlying send
        // and fold it up appropriately.
        if (completedSend instanceof LazyDownConversionRecordsSend) {
            if (recordConversionStats == null)
                recordConversionStats = new HashMap<>();
            LazyDownConversionRecordsSend lazyRecordsSend = (LazyDownConversionRecordsSend) completedSend;
            recordConversionStats.put(lazyRecordsSend.topicPartition(), lazyRecordsSend.recordConversionStats());
        }
    }
}
```

但是我们还是要问数据在哪？

- 你可以说，它在`Send send`里面。但是这货是一个接口啊
- 所以说，数据在Send接口的具体实现中，即：ByteBufferSend和MultiRecordsSend

在NetworkClient的doSend()方法的最后一行，`selector.send(new NetworkSend(clientRequest.destination(), send))`利用持有数据的send构造了一个新的`NetworkSend`，然后用selector发送出去了。

## NetworkSend

```java
public class NetworkSend implements Send {
    private final String destinationId;
    private final Send send;

    public NetworkSend(String destinationId, Send send) {
        this.destinationId = destinationId;
        this.send = send;
    }

    public String destinationId() {
        return destinationId;
    }

    @Override
    public boolean completed() {
        return send.completed();
    }

    @Override
    public long writeTo(TransferableChannel channel) throws IOException {
        return send.writeTo(channel);
    }

    @Override
    public long size() {
        return send.size();
    }

}
```

NetworkSend作为Send的一个实现类，只是包装了一层destinationId而已。

**接下来就开始看TransferableChannel和Selector部分，他们如何同Send配合来把数据发送出去**

## TransferableChannel

```java
public interface TransferableChannel extends GatheringByteChannel {

  /**
   * @return true if there are any pending writes. false if the implementation directly write all data to output.
   */
  boolean hasPendingWrites();

  /**
   * Transfers bytes from `fileChannel` to this `TransferableChannel`.
   *
   * This method will delegate to {@link FileChannel#transferTo(long, long, java.nio.channels.WritableByteChannel)},
   * but it will unwrap the destination channel, if possible, in order to benefit from zero copy. This is required
   * because the fast path of `transferTo` is only executed if the destination buffer inherits from an internal JDK
   * class.
   *
   * @param fileChannel The source channel
   * @param position The position within the file at which the transfer is to begin; must be non-negative
   * @param count The maximum number of bytes to be transferred; must be non-negative
   * @return The number of bytes, possibly zero, that were actually transferred
   * @see FileChannel#transferTo(long, long, java.nio.channels.WritableByteChannel)
   */
  long transferFrom(FileChannel fileChannel, long position, long count) throws IOException;
}
```

它继承了GatheringByteChannel

```java
public interface GatheringByteChannel
    extends WritableByteChannel
{
	public long write(ByteBuffer[] srcs, int offset, int length)	throws IOException;
	public long write(ByteBuffer[] srcs) throws IOException;
};
```

GatheringByteChannel是一个java的nio包中的channel，里面有write方法把数据写到nio的Channel中。

**这里要改成用RDMA传输的话，需要考虑一下怎么把数据给搞过来：**

- 方法一：改一下Send接口，添加一个返回ByteBuffer和数据长度的方法。然后还要改Send接口的具体实现。但是这个方法的难点在于具体实现时怎么改？
- 方法二：自己定义一个RdmaChannel，重写write方法，改成把数据写到一个自己定义的内存中去。

个人倾向于方法二。

数据在哪的问题暂且告一段落，接下来关注Selector如何把数据发送出去。

## Selector

Selector是一个实现了selectable接口的类。

里面有太多的方法了，我们先看send()方法：

```java
/**
 * Queue the given request for sending in the subsequent {@link #poll(long)} calls
 * @param send The request to send
 */
public void send(NetworkSend send) {
    String connectionId = send.destinationId();
    KafkaChannel channel = openOrClosingChannelOrFail(connectionId);
    if (closingChannels.containsKey(connectionId)) {
        // ensure notification via `disconnected`, leave channel in the state in which closing was triggered
        this.failedSends.add(connectionId);
    } else {
        try {
            channel.setSend(send);// 调用了KafkaChannel的setSend()
        } catch (Exception e) {
            // update the state for consistency, the channel will be discarded after `close`
            channel.state(ChannelState.FAILED_SEND);
            // ensure notification via `disconnected` when `failedSends` are processed in the next poll
            this.failedSends.add(connectionId);
            close(channel, CloseMode.DISCARD_NO_NOTIFY);
            if (!(e instanceof CancelledKeyException)) {
                log.error("Unexpected exception during send, closing connection {} and rethrowing exception {}",
                        connectionId, e);
                throw e;
            }
        }
    }
}
```

send()方法只是调用了channel.setSend()，并没有真的把数据发送出去。

poll()函数：

```java
/**
 * Do whatever I/O can be done on each connection without blocking. This includes completing connections, completing
 * disconnections, initiating new sends, or making progress on in-progress sends or receives.
 *
 * When this call is completed the user can check for completed sends, receives, connections or disconnects using
 * {@link #completedSends()}, {@link #completedReceives()}, {@link #connected()}, {@link #disconnected()}. These
 * lists will be cleared at the beginning of each `poll` call and repopulated by the call if there is
 * any completed I/O.
 *
 * In the "Plaintext" setting, we are using socketChannel to read & write to the network. But for the "SSL" setting,
 * we encrypt the data before we use socketChannel to write data to the network, and decrypt before we return the responses.
 * This requires additional buffers to be maintained as we are reading from network, since the data on the wire is encrypted
 * we won't be able to read exact no.of bytes as kafka protocol requires. We read as many bytes as we can, up to SSLEngine's
 * application buffer size. This means we might be reading additional bytes than the requested size.
 * If there is no further data to read from socketChannel selector won't invoke that channel and we have additional bytes
 * in the buffer. To overcome this issue we added "keysWithBufferedRead" map which tracks channels which have data in the SSL
 * buffers. If there are channels with buffered data that can by processed, we set "timeout" to 0 and process the data even
 * if there is no more data to read from the socket.
 *
 * Atmost one entry is added to "completedReceives" for a channel in each poll. This is necessary to guarantee that
 * requests from a channel are processed on the broker in the order they are sent. Since outstanding requests added
 * by SocketServer to the request queue may be processed by different request handler threads, requests on each
 * channel must be processed one-at-a-time to guarantee ordering.
 *
 * @param timeout The amount of time to wait, in milliseconds, which must be non-negative
 * @throws IllegalArgumentException If `timeout` is negative
 * @throws IllegalStateException If a send is given for which we have no existing connection or for which there is
 *         already an in-progress send
 */
@Override
public void poll(long timeout) throws IOException {
    if (timeout < 0)
        throw new IllegalArgumentException("timeout should be >= 0");

    boolean madeReadProgressLastCall = madeReadProgressLastPoll;
    clear();

    boolean dataInBuffers = !keysWithBufferedRead.isEmpty();

    if (!immediatelyConnectedKeys.isEmpty() || (madeReadProgressLastCall && dataInBuffers))
        timeout = 0;

    if (!memoryPool.isOutOfMemory() && outOfMemory) {
        //we have recovered from memory pressure. unmute any channel not explicitly muted for other reasons
        log.trace("Broker no longer low on memory - unmuting incoming sockets");
        for (KafkaChannel channel : channels.values()) {
            if (channel.isInMutableState() && !explicitlyMutedChannels.contains(channel)) {
                channel.maybeUnmute();
            }
        }
        outOfMemory = false;
    }

    /* check ready keys */
    long startSelect = time.nanoseconds();
    int numReadyKeys = select(timeout);// 选出准备好的keys
    long endSelect = time.nanoseconds();
    this.sensors.selectTime.record(endSelect - startSelect, time.milliseconds());

    if (numReadyKeys > 0 || !immediatelyConnectedKeys.isEmpty() || dataInBuffers) {
        Set<SelectionKey> readyKeys = this.nioSelector.selectedKeys();

        // Poll from channels that have buffered data (but nothing more from the underlying socket)
        if (dataInBuffers) {
            keysWithBufferedRead.removeAll(readyKeys); //so no channel gets polled twice
            Set<SelectionKey> toPoll = keysWithBufferedRead;
            keysWithBufferedRead = new HashSet<>(); //poll() calls will repopulate if needed
            pollSelectionKeys(toPoll, false, endSelect);
        }

        // Poll from channels where the underlying socket has more data
        pollSelectionKeys(readyKeys, false, endSelect);
        // Clear all selected keys so that they are included in the ready count for the next select
        readyKeys.clear();

        pollSelectionKeys(immediatelyConnectedKeys, true, endSelect);
        immediatelyConnectedKeys.clear();
    } else {
        madeReadProgressLastPoll = true; //no work is also "progress"
    }

    long endIo = time.nanoseconds();
    this.sensors.ioTime.record(endIo - endSelect, time.milliseconds());

    // Close channels that were delayed and are now ready to be closed
    completeDelayedChannelClose(endIo);

    // we use the time at the end of select to ensure that we don't close any connections that
    // have just been processed in pollSelectionKeys
    maybeCloseOldestConnection(endSelect);
}
```

poll()函数是非阻塞的。

```java
/**
 * Check for data, waiting up to the given timeout.
 *
 * @param timeoutMs Length of time to wait, in milliseconds, which must be non-negative
 * @return The number of keys ready
 */
private int select(long timeoutMs) throws IOException {
    if (timeoutMs < 0L)
        throw new IllegalArgumentException("timeout should be >= 0");

    if (timeoutMs == 0L)
        return this.nioSelector.selectNow();
    else
        return this.nioSelector.select(timeoutMs);
}
```



```java
/**
 * handle any ready I/O on a set of selection keys
 * @param selectionKeys set of keys to handle
 * @param isImmediatelyConnected true if running over a set of keys for just-connected sockets
 * @param currentTimeNanos time at which set of keys was determined
 */
// package-private for testing
void pollSelectionKeys(Set<SelectionKey> selectionKeys,
                       boolean isImmediatelyConnected,
                       long currentTimeNanos) {
    for (SelectionKey key : determineHandlingOrder(selectionKeys)) {
        KafkaChannel channel = channel(key);
        long channelStartTimeNanos = recordTimePerConnection ? time.nanoseconds() : 0;
        boolean sendFailed = false;
        String nodeId = channel.id();

        // register all per-connection metrics at once
        sensors.maybeRegisterConnectionMetrics(nodeId);
        if (idleExpiryManager != null)
            idleExpiryManager.update(nodeId, currentTimeNanos);

        try {
            /* complete any connections that have finished their handshake (either normally or immediately) */
            if (isImmediatelyConnected || key.isConnectable()) {
                if (channel.finishConnect()) {
                    this.connected.add(nodeId);
                    this.sensors.connectionCreated.record();

                    SocketChannel socketChannel = (SocketChannel) key.channel();
                    log.debug("Created socket with SO_RCVBUF = {}, SO_SNDBUF = {}, SO_TIMEOUT = {} to node {}",
                            socketChannel.socket().getReceiveBufferSize(),
                            socketChannel.socket().getSendBufferSize(),
                            socketChannel.socket().getSoTimeout(),
                            nodeId);
                } else {
                    continue;
                }
            }

            /* if channel is not ready finish prepare */
            if (channel.isConnected() && !channel.ready()) {
                channel.prepare();
                if (channel.ready()) {
                    long readyTimeMs = time.milliseconds();
                    boolean isReauthentication = channel.successfulAuthentications() > 1;
                    if (isReauthentication) {
                        sensors.successfulReauthentication.record(1.0, readyTimeMs);
                        if (channel.reauthenticationLatencyMs() == null)
                            log.warn(
                                "Should never happen: re-authentication latency for a re-authenticated channel was null; continuing...");
                        else
                            sensors.reauthenticationLatency
                                .record(channel.reauthenticationLatencyMs().doubleValue(), readyTimeMs);
                    } else {
                        sensors.successfulAuthentication.record(1.0, readyTimeMs);
                        if (!channel.connectedClientSupportsReauthentication())
                            sensors.successfulAuthenticationNoReauth.record(1.0, readyTimeMs);
                    }
                    log.debug("Successfully {}authenticated with {}", isReauthentication ?
                        "re-" : "", channel.socketDescription());
                }
            }
            if (channel.ready() && channel.state() == ChannelState.NOT_CONNECTED)
                channel.state(ChannelState.READY);
            Optional<NetworkReceive> responseReceivedDuringReauthentication = channel.pollResponseReceivedDuringReauthentication();
            responseReceivedDuringReauthentication.ifPresent(receive -> {
                long currentTimeMs = time.milliseconds();
                addToCompletedReceives(channel, receive, currentTimeMs);
            });

            //if channel is ready and has bytes to read from socket or buffer, and has no
            //previous completed receive then read from it
            if (channel.ready() && (key.isReadable() || channel.hasBytesBuffered()) && !hasCompletedReceive(channel)
                    && !explicitlyMutedChannels.contains(channel)) {
                attemptRead(channel);
            }

            if (channel.hasBytesBuffered()) {
                //this channel has bytes enqueued in intermediary buffers that we could not read
                //(possibly because no memory). it may be the case that the underlying socket will
                //not come up in the next poll() and so we need to remember this channel for the
                //next poll call otherwise data may be stuck in said buffers forever. If we attempt
                //to process buffered data and no progress is made, the channel buffered status is
                //cleared to avoid the overhead of checking every time.
                keysWithBufferedRead.add(key);
            }

            /* if channel is ready write to any sockets that have space in their buffer and for which we have data */

            long nowNanos = channelStartTimeNanos != 0 ? channelStartTimeNanos : currentTimeNanos;
            try {
                attemptWrite(key, channel, nowNanos);// 尝试写
            } catch (Exception e) {
                sendFailed = true;
                throw e;
            }

            /* cancel any defunct sockets */
            if (!key.isValid())
                close(channel, CloseMode.GRACEFUL);

        } catch (Exception e) {
            String desc = channel.socketDescription();
            if (e instanceof IOException) {
                log.debug("Connection with {} disconnected", desc, e);
            } else if (e instanceof AuthenticationException) {
                boolean isReauthentication = channel.successfulAuthentications() > 0;
                if (isReauthentication)
                    sensors.failedReauthentication.record();
                else
                    sensors.failedAuthentication.record();
                String exceptionMessage = e.getMessage();
                if (e instanceof DelayedResponseAuthenticationException)
                    exceptionMessage = e.getCause().getMessage();
                log.info("Failed {}authentication with {} ({})", isReauthentication ? "re-" : "",
                    desc, exceptionMessage);
            } else {
                log.warn("Unexpected error from {}; closing connection", desc, e);
            }

            if (e instanceof DelayedResponseAuthenticationException)
                maybeDelayCloseOnAuthenticationFailure(channel);
            else
                close(channel, sendFailed ? CloseMode.NOTIFY_ONLY : CloseMode.GRACEFUL);
        } finally {
            maybeRecordTimePerConnection(channel, channelStartTimeNanos);
        }
    }
}
```

里面调用了attemptWrite()尝试写数据:

```java
private void attemptWrite(SelectionKey key, KafkaChannel channel, long nowNanos) throws IOException {
    if (channel.hasSend()
            && channel.ready()
            && key.isWritable()
            && !channel.maybeBeginClientReauthentication(() -> nowNanos)) {
        write(channel);
    }
}
```

```java
void write(KafkaChannel channel) throws IOException {
    String nodeId = channel.id();
    long bytesSent = channel.write();// 调用了channel.write()
    NetworkSend send = channel.maybeCompleteSend();
    // We may complete the send with bytesSent < 1 if `TransportLayer.hasPendingWrites` was true and `channel.write()`
    // caused the pending writes to be written to the socket channel buffer
    if (bytesSent > 0 || send != null) {
        long currentTimeMs = time.milliseconds();
        if (bytesSent > 0)
            this.sensors.recordBytesSent(nodeId, bytesSent, currentTimeMs);
        if (send != null) {
            this.completedSends.add(send);
            this.sensors.recordCompletedSend(nodeId, send.size(), currentTimeMs);
        }
    }
}
```

所以最后的write是调用了KafkaChannel的write()。

## KafkaChannel

```java
public class KafkaChannel implements AutoCloseable {
    private final String id;
    private final TransportLayer transportLayer;
    private final Supplier<Authenticator> authenticatorCreator;
    private Authenticator authenticator;
    // Tracks accumulated network thread time. This is updated on the network thread.
    // The values are read and reset after each response is sent.
    private long networkThreadTimeNanos;
    private final int maxReceiveSize;
    private final MemoryPool memoryPool;
    private final ChannelMetadataRegistry metadataRegistry;
    private NetworkReceive receive;
    private NetworkSend send;// 发送的数据的引用
    // Track connection and mute state of channels to enable outstanding requests on channels to be
    // processed after the channel is disconnected.
    private boolean disconnected;
    private ChannelMuteState muteState;
    private ChannelState state;
    private SocketAddress remoteAddress;
    private int successfulAuthentications;
    private boolean midWrite;
    private long lastReauthenticationStartNanos;
  ...
}    
```



```java
public void setSend(NetworkSend send) {
    if (this.send != null)
        throw new IllegalStateException("Attempt to begin a send operation with prior send operation still in progress, connection id is " + id);
    this.send = send;
    this.transportLayer.addInterestOps(SelectionKey.OP_WRITE);
}
```

```java
public long write() throws IOException {
    if (send == null)
        return 0;

    midWrite = true;
    return send.writeTo(transportLayer);
}
```

行吧，这里是让拥有数据的send把数据写入到transportLayer里面。

## TransportLayer

是一个接口，继承了TransferableChannel

```java
public interface TransportLayer extends ScatteringByteChannel, TransferableChannel{
	boolean ready();
  /**
   * returns underlying socketChannel
   */
  SocketChannel socketChannel();

  /**
   * Get the underlying selection key
   */
  SelectionKey selectionKey();
  ...
}
```

它的实现有`PlainTextTransportLayer`和`SslTransportLayer`

```java
public class PlaintextTransportLayer implements TransportLayer {
  private final SelectionKey key;
  private final SocketChannel socketChannel;
  private final Principal principal = KafkaPrincipal.ANONYMOUS;
  public PlaintextTransportLayer(SelectionKey key) throws IOException {
      this.key = key;
      this.socketChannel = (SocketChannel) key.channel();
  }
  ...
};
```

write()如下：

```java
/**
* Writes a sequence of bytes to this channel from the given buffer.
*
* @param src The buffer from which bytes are to be retrieved
* @return The number of bytes read, possibly zero, or -1 if the channel has reached end-of-stream
* @throws IOException If some other I/O error occurs
*/
@Override
public int write(ByteBuffer src) throws IOException {
    return socketChannel.write(src);
}

/**
* Writes a sequence of bytes to this channel from the given buffer.
*
* @param srcs The buffer from which bytes are to be retrieved
* @return The number of bytes read, possibly zero, or -1 if the channel has reached end-of-stream
* @throws IOException If some other I/O error occurs
*/
@Override
public long write(ByteBuffer[] srcs) throws IOException {
    return socketChannel.write(srcs);
}
```

SelectionKey是Java内部的接口