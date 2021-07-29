# Producer

目前代码都是在`clients/src/main`下的。

位置`java/org/apache/kafka/clients/producer/KafkaProducer.java`

Producer会调用`send()`发送ProducerRecord，并返回`Future<RecordMetadata>`。
而`send()`函数底层调用`dosend()`来实现异步发送ProducerRecord。

`send()`的一个参数是一个callback函数。Kafka在返回响应时会调用callback函数来实现异步的发送确认。这个callback函数要实现`onCompletion(RecordMetadata, Exception)`方法，这样用户能定义在发送成功/失败时的操作。对于同一个partition，如果record1先于record2发送，那么callback1会先于callback2调用。

Producer发送完会调用`close()`回收资源。

在发送之前进行序列化（这个可以考虑优化掉？）。
序列化后的消息会先放到一个队列里面，应该是为了组成一个batch，一次发送。

然后使用this.sender.wakeup()来发送消息。

ProducerRecord的位置是`java/org/apache/kafka/clients/producer/ProducerRecord.java`。

``` java
public class ProducerRecord<K, V> {

    private final String topic;
    private final Integer partition;
    private final Headers headers;
    private final K key;
    private final V value;
    private final Long timestamp;
};
```

RecordMetedata包含一些消息的元数据：

位置是`java/org/apache/kafka/clients/producer/RecordMetadata.java`

``` java
public final class RecordMetadata {

    /**
     * Partition value for record without partition assigned
     */
    public static final int UNKNOWN_PARTITION = -1;

    private final long offset;
    // The timestamp of the message.
    // If LogAppendTime is used for the topic, the timestamp will be the timestamp returned by the broker.
    // If CreateTime is used for the topic, the timestamp is the timestamp in the corresponding ProducerRecord if the
    // user provided one. Otherwise, it will be the producer local time when the producer record was handed to the
    // producer.
    private final long timestamp;
    private final int serializedKeySize;
    private final int serializedValueSize;
    private final TopicPartition topicPartition;

    private volatile Long checksum;
};
```

发消息的三种方式：

- 发后即忘(fire-and-forget)
- 同步(aync)
- 异步(async)

## 序列化

Serializer是接口，具体是实现是可以配置的，然后用反射读出来。

在`java/org/apache/kafka/common/serialization`下有很多种Serializer。

## 分区器 Partitioner

如果ProducerRecord指定了partition字段，就不需要Partitioner了，否则Partitioner会根据key值来计算partition的值，从而为消息分区。

Partitioner接口：

- 位置`java/org/apache/kafka/clients/producer/Partitioner.java`

默认分区器:DefaultPartitioner

- 位置：`java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java`
- 把key值做Murmur2哈希来分区

## 拦截器 Interceptor

ProducerInterceptor:

- 一个接口：
  - 在`dosend()`之前进行一些准备工作，会调用`onSend()`。
  - 在发送的消息被确认后，或是在发送前就失败时。会调用`onAcknowledgement()`方法。一般会在调用用户的`callback()`之前调用`onAcknowledgement()`
- 位置：`java/org/apache/kafka/clients/producer/internals/ProducerInterceptor.java`

``` java
public ProducerRecord<K, V> onSend(ProducerRecord<K, V> record);
public void onAcknowledgement(RecordMetadata metadata, Exception exception);
public void close();
```

不仅能指定一个Interceptor，还可以指定Interceptor chain。

## RecordAccumulator

- 用来缓存消息，使Sender线程能够批量发送，减少网络传输的资源消耗
- 位置：
`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\clients\producer\internals\RecordAccumulator.java`
- 缓存大小通过`buffer.memory`设置
- 为每个partition都维护了一个deque，`Deque<ProducerBatch>`。producer产生的消息写道尾部，sender从头部读消息。
- 在发送之前要创建一块内存区域保存对应的消息。在client中是通过java.io.ByteBuffer实现的。但是频繁创建ByteBuffer消耗资源，所以RecordAccumulator内部还有一个BufferPool。
- Sender从RecordAccumulator获取消息后，会将`<Partition, Deque<ProducerBatch>>`形式转换为`<Node, List<ProducerBatch>>`形式(Node对应broker节点）。因为sender只关心发往哪个节点，而不关心消息的partition。
- 之后sender会把`<Node, List<ProducerBatch>>`进一步封装为`<Node, Request>`的形式。然后将Request发往各个Node。
- 把request从sender发送之前还会保存到InFlightRequest中，它主要保存已经发出去但是还没收到响应的请求。InFlightRequest可以配置缓存数等。