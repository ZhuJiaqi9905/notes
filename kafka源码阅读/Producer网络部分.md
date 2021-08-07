# Producer网络部分

Producer的实现类是`KafkaProducer`

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
                compressionType, serializedKey, serializedValue, headers);
        ensureValidRecordSize(serializedSize);
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

        if (result.abortForNewBatch) {
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
            this.sender.wakeup();
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