# Disni

Api : https://www.ibm.com/docs/en/sdk-java-technology/7.1?topic=reference-networking-jverbs

一些术语：https://www.ibm.com/docs/en/sdk-java-technology/7.1?topic=library-jverbs-programming-terms-artifacts

可以学习一下源码中的类的设计

`class IbvSge`

- 就是SG表的元素，加上getter和setter

`class IbvWC`

- 就是wc，加上getter和setter
- 在c语言中,一般都是在栈上分配wc，然后作为ibv_poll_cq的出参。也可以预先分配好然后全局重用

`RdmaConnParam`

- rdma connection management的参数



`RdmaCm`

- Rdma connection management

- ```java
  public abstract RdmaEventChannel createEventChannel() throws IOException;
  public RdmaCmId createId(RdmaEventChannel cmChannel, short rdma_ps)
  public IbvQP createQP(RdmaCmId id, IbvPd pd, IbvQPInitAttr attr)
  public void bindAddr(RdmaCmId id, SocketAddress addr)
  public abstract void listen(RdmaCmId id, int backlog)
  public abstract void resolveAddr(RdmaCmId id, SocketAddress src, SocketAddress dst, int timeout)
  public abstract void resolveRoute(RdmaCmId id, int timeout)
  			throws IOException;
  public abstract RdmaCmEvent getCmEvent(RdmaEventChannel cmChannel, int timeout)
  			throws IOException;
  public abstract void connect(RdmaCmId id, RdmaConnParam connParam)
  			throws IOException;
  public abstract void accept(RdmaCmId id, RdmaConnParam connParam)
  			throws IOException;
  public abstract int ackCmEvent(RdmaCmEvent cmEvent);
  public abstract int disconnect(RdmaCmId id) throws IOException;
  public abstract int destroyEventChannel(RdmaEventChannel cmChannel) throws IOException;
  public abstract int destroyCmId(RdmaCmId id) throws IOException;
  public abstract int destroyQP(RdmaCmId id) throws IOException;
  public abstract SocketAddress getSrcAddr(RdmaCmId id) throws IOException;
  public abstract SocketAddress getDstAddr(RdmaCmId id) throws IOException;
  public abstract int destroyEp(RdmaCmId id) throws IOException;
  ```

`RdmaCmEvent`

- 对cmEvent的封装

`RdmaEventChannel`

- ```
  	private RdmaCm cm;
  	private int fd;
  	protected volatile boolean isOpen;
  ```

- 只有上面这几个成员变量，它的操作都是靠RdmaCm完成的。

- 感觉我们设计的时候可以改成让RdmaEventChannel单独维护自己的创建和销毁

## RdmaEventChannel

`createEventChannel()`

```java
/**
 * Asynchronous events are reported to users through event channels.
 * 
 * Event channels are used to direct all events on an RdmaCmId. For many clients, a single event channel may be sufficient, however, when managing a large number of connections or id's. users may find it useful to direct events for different id's to different channels for processing.
 * 
 * All created event channels must be destroyed by calling destroyEventChannel. Users should call getCmEvent to retrieve events on an event channel.
 *
 * @return the rdma event channel
 * @throws Exception the exception
 */
public abstract RdmaEventChannel createEventChannel() throws IOException;
```

- 要调用`destroyEventChannel()`销毁



`createId()`

```java
/**
 * Creates an identifier that is used to track communication information.
 * 
 * RdmaCmId's are conceptually equivalent to a socket for RDMA communication.  The difference is that RDMA communication requires explicitly binding to a specified RDMA device before communication can occur, and most operations are asynchronous in nature.  Asynchronous communication events on an id are reported through the associated event channel.  
 * 
 * Users must release the id by calling destroyCmId.
 *
 * @param cmChannel the communication channel that events associated with the allocated RdmaCmId will be reported on.
 * @param rdma_ps the RDMA port space. Only RDMA_PS_TCP supported.
 * @return the newly created Id 
 * @throws Exception on failure.
 */
public abstract RdmaCmId createId(RdmaEventChannel cmChannel, short rdma_ps) throws IOException;

```

- 创建一个用于追踪连接信息的标识
- 等同于一个socket

## RdmaCm

是RdmaCmEvent内部持有的类。调用时候都是调用RdmaCmEvent的同名方法

```java
/**
 * Retrieves a communication event. If no events are pending, by default, the call will block until an event is received.
 * 
 * All events that are reported must be acknowledged by calling ackCmEvent.
 *
 * @param cmChannel event channel to check for events.
 * @return information about the next communication event.
 * @throws Exception on failure.
 */
public abstract RdmaCmEvent getCmEvent(RdmaEventChannel cmChannel, int timeout)
    throws IOException;

```



```java
/**
 * All events which are allocated by getCmEvent must be released, there should be a one-to-one correspondence between successful gets and acks.
 *
 * @param cmEvent the event to be released.
 * @return return 0 on success, 1 on failure.
 */
public abstract int ackCmEvent(RdmaCmEvent cmEvent);
```

