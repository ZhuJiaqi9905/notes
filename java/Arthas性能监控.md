# Arthas性能监控

https://arthas.aliyun.com/doc/index.html#



下载和启动：

```
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

## dashboard

输入`dashboard`按`回车/enter`，会展示当前进程的信息。按`ctrl+c`可以中断执行。

### 数据说明

- ID: Java级别的线程ID，注意这个ID不能跟jstack中的nativeID一一对应。
- NAME: 线程名
- GROUP: 线程组名
- PRIORITY: 线程优先级, 1~10之间的数字，越大表示优先级越高
- STATE: 线程的状态
- CPU%: 线程的cpu使用率。比如采样间隔1000ms，某个线程的增量cpu时间为100ms，则cpu使用率=100/1000=10%
- DELTA_TIME: 上次采样之后线程运行增量CPU时间，数据格式为`秒`
- TIME: 线程运行总CPU时间，数据格式为`分:秒`
- INTERRUPTED: 线程当前的中断位状态
- DAEMON: 是否是daemon线程

### JVM内部线程

- 当JVM 堆(heap)/元数据(metaspace)空间不足或OOM时，可以看到GC线程的CPU占用率明显高于其他的线程。
- 当执行`trace/watch/tt/redefine`等命令后，可以看到JIT线程活动变得更频繁。因为JVM热更新class字节码时清除了此class相关的JIT编译结果，需要重新编译。

JVM内部线程包括下面几种：

- JIT编译线程: 如 `C1 CompilerThread0`, `C2 CompilerThread0`
- GC线程: 如`GC Thread0`, `G1 Young RemSet Sampling`
- 其它内部线程: 如`VM Periodic Task Thread`, `VM Thread`, `Service Thread`

## thread

`thread i`会打印线程i的栈。

## 通过jad来反编译

`jad demo.MathGame`



## watch

通过`watch`命令来查看`demo.MathGame#primeFactors`函数的返回值：

## 退出

退出当前的连接：用`quit`或者`exit`命令。Attach到目标进程上的arthas还会继续运行，端口会保持开放，下次连接时可以直接连接上。

完全退出arthas：执行`stop`命令。



vmoption

perfcounter

sc

sm

vmtool

trace

- 打印链路的耗时

- https://arthas.aliyun.com/doc/trace.html

stack

- 输出当前方法被调用的调用路径
- https://arthas.aliyun.com/doc/stack.html

tt

- 方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测
- https://arthas.aliyun.com/doc/tt.html

profiler

- 支持生成应用热点的火焰图
- https://arthas.aliyun.com/doc/profiler.html

