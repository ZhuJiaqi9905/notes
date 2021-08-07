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