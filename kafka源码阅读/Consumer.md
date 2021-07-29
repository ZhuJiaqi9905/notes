# Consumer

## TopicPartition

TopicPartition类

- 用来表示分区: 主题和partition编号
- 位置：`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\common\TopicPartition.java`

``` java
public final class TopicPartition implements Serializable {
    private static final long serialVersionUID = -613627415771699627L;

    private int hash = 0;
    private final int partition;
    private final String topic;
};
```

## PartitionInfo

PartitionInfo

- 用来表示partition的元数据信息
- 位置：`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\common\PartitionInfo.java`
- KafkaConsumer中的partitionFor()方法能查询指定主题的元数据信息

``` java
public class PartitionInfo {
    private final String topic;
    private final int partition;
    private final Node leader;
    private final Node[] replicas; // AR
    private final Node[] inSyncReplicas; // ISR
    private final Node[] offlineReplicas; // OSR
};
```

- AR(Assigned Replicas): 分区的所有副本
- ISR(In-Sync Replicas): 与leader副本保持一定程度同步的所有副本
- OSR(Out-of-Sync Replicas): 与leader副本同步之后过多的副本（不包括leader副本）
- leader副本负责维护和跟踪ISR的之后状态，如果某个副本的滞后太严重，就把它从ISR移除，加入到OSR中

## Deserializer

Deserializer

- 是一个接口。用来把data进行反序列化
- 位置：`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\common\serialization\Deserializer.java`
- 有很多的实现类


## KafkaConsumer

Consumer

- 是一个接口
- 位置：`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\clients\consumer\Consumer.java`
- 比较重要的方法

``` java
void subscribe(Collection<String> topics, ConsumerRebalanceListener callback);
void subscribe(Collection<String> topics);
void subscribe(Pattern pattern, ConsumerRebalanceListener callback);
void subscribe(Pattern pattern);
void unsubscribe();
ConsumerRecords<K, V> poll(Duration timeout);
```

KafkaConsumer

- 是Consumer的一个实现类
- 是kafka中用来消费消息的消费者

### 消息消费

kafka使用pull模式来消费消息。具体地，就是用`poll()`方法来定时轮询，已得到可以消费的消息。


## ConsumerRecord

位置：`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\clients\consumer\ConsumerRecord.java`
``` java
public class ConsumerRecord<K, V> {
    public static final long NO_TIMESTAMP = RecordBatch.NO_TIMESTAMP;
    public static final int NULL_SIZE = -1;
    public static final int NULL_CHECKSUM = -1;

    private final String topic;
    private final int partition;
    private final long offset;
    private final long timestamp;
    private final TimestampType timestampType;
    private final int serializedKeySize;
    private final int serializedValueSize;
    private final Headers headers;// 消息的头部
    private final K key;
    private final V value;
    private final Optional<Integer> leaderEpoch;

    private volatile Long checksum;
};
```

## 位移提交

- （消费）位移：消费者消费到的位置
- 偏移量：消息在分区中的位置

消费者要把消费位移持久化保存。每次调用`poll()`时，消费者才能得到没被消费过的消息集。把消费位移存储起来的动作称为“提交”，消费者在消费完消息后需要提交消费位移。

- 提交的是下一条需要拉取的消息的位置。
- 消息提交的具体时机不同，可能导致消息丢失或重复消费的情况发生。kafka默认的提交方式是自动提交，并且是定时提交。它无法避免消息丢失或者重复消费。
- kakfa还提供了手动位移提交的方式。一般开发人员拉取消息后，还要进行复杂的业务逻辑。开发人员要根据业务逻辑来选择合适的地方进行手动位移提交。
  - 同步提交：`commitSync()`
  - 异步提交：`commitAsync()`
  - 实际应用中很少有“消费一条消息就进行一次提交”的情况。一般会批量或是按照分区粒度。
  - 异步提交时，如果出现一次失败，要考虑是否需要再提交的问题（如果你的提交把更大的位移覆盖了，会出现重复消费）。可以设置一个递增序号来维护异步提交的顺序。

## 控制或关闭消费

- 暂停/恢复对某些分区的消费

``` java
public void pause(Collection<TopicPartition> partitions);
public void resume(Collection<TopicPartition> partitions);
public Set<TopicPartition> paused();//返回已经被暂停的分区
```

- 关闭并释放资源

``` java
public void close();
```

## 指定位移来进行消费

``` java
public void seek(TopicPartition partition, long offset);
```

如果poll()方法中的参数设置为0，那么此方法立刻返回，poll()内部进行分区分配的逻辑就来不及实施。那么此时的消费者并未分配到任何分区，即`consumer.assignment()`返回一个空表。

## Rebalance 再均衡

Rebalance指分区的所属权从一个消费者转移到另一个消费者的行为。它为消费组具备高可用性和伸缩性提供保障。

- 允许我们能往消费组里面添加/删除消费者
- rebalance期间，消费组里面的消费者是无法读取消息的

ConsumerRebalanceListener

- 是一个接口
- 位置：`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\clients\consumer\ConsumerRebalanceListener.java`

``` java
//  rebalance开始前和消费者停止读取消息后被调用。
// 可以通过它来处理消费位移的提交，来避免一些不必要的重复消费现象的发生。
void onPartitionsRevoked(Collection<TopicPartition> partitions);
// 在重新分配分区后和消费则开始读取消费之前被调用。
void onPartitionsAssigned(Collection<TopicPartition> partitions);
```

## Interceptor拦截器

和生产者拦截器类似，消费者拦截器主要在消费到消息或在提交消费位移时进行定制化操作。

ConsumerInterceptor

- 是一个接口
- 位置：`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\clients\consumer\ConsumerInterceptor.java`

``` java
public ConsumerRecords<K, V> onConsume(ConsumerRecords<K, V> records);
public void onCommit(Map<TopicPartition, OffsetAndMetadata> offsets);
public void close();
```

- 在`poll()`方法返回之前调用`onConsume()`对消息进行定制化操作
- 在提交消费位移后调用`onCommit()`方法
