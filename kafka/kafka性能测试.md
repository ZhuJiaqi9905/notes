# kafka性能测试

在学习了一学期kafka并阅读了部分源码后，对kafka有了更多的理解。重新进行性能测试并记录于此。

测试版本：kafka2.8，使用zookeeper

对kafka性能比较重要的参数：

- java堆内存

- 是否开启ack
- 消息大小
- 是否压缩及压缩算法

## 单producer单broker

测试脚本在`/bin/kafka-producer-perf-test.sh`中，大致内容就是发送key和value都是`Byte[]`的消息，测试吞吐量。

### 测试1

```
./bin/kafka-producer-perf-test.sh --topic test-topic --num-records 50000000 --throughput -1 --record-size 1024 --producer-props bootstrap.servers=10.0.12.24:9988  acks=-1 linger.ms=2000 compression.type=lz4
```

- 不启用ack
- 使用lz4压缩
- `linger.ms=2000`

结果：

```
1097804.1 records/sec (1072.07 MB/sec), 0.8 ms avg latency, 35.0 ms max latency.
```

吞吐量约1GB/s，平均延迟在0.8～1.5ms内

### 测试2

```
./bin/kafka-producer-perf-test.sh --topic test-topic --num-records 50000000 --throughput -1 --record-size 1024 --producer-props bootstrap.servers=10.0.12.24:9988  acks=-1 linger.ms=2000 compression.type=none
```

- 不启用ack
- 不使用压缩
- `linger.ms=2000`

结果：

```
328980 records sent, 54394.8 records/sec (53.12 MB/sec), 243.5 ms avg latency, 3661.0 ms max latency.
```

```
524730 records sent, 104946.0 records/sec (102.49 MB/sec), 493.8 ms avg latency, 3797.0 ms max latency.
```

```
483165 records sent, 96633.0 records/sec (94.37 MB/sec), 319.4 ms avg latency, 1311.0 ms max latency.
```

不启用压缩后，结果只有100MB/s左右。平均延迟在300ms左右。

## 单consumer单broker

### 测试1

```
./bin/kafka-consumer-perf-test.sh --broker-list 10.0.12.24:9988 --messages 50000000 --topic test-topic
```

结果：

```
1012.1169MB/s
```

