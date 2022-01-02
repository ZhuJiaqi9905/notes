客户端的网络传输：
KafkaProducer使用了Sender类的send()方法传输。

Sender类主要是封装Requeest，然后底层调用KafkaClient接口的send()方法传输。

KafkaClient是一个接口，它的实现是NetworkClient类。

NetworkClient类是调用Selectable接口的send方法来发送数据。

Selectable接口的具体实现是Selector类。
位置是`D:\Code\4_Scala\kafka-2.8\clients\src\main\java\org\apache\kafka\common\network\Selector.java`