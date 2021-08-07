# metadata

`kafka-2.8\metadata`

## controller

`D:\Code\4_Scala\kafka-2.8\metadata\src\main\java\org\apache\kafka\controller`

主要是控制broker，进行主从选举之类的。

### BrokerHeartbeatManager

The BrokerHeartbeatManager manages all the soft state associated with broker heartbeats.

Soft state is state which does not appear in the metadata log.
This state includes things like the last time each broker sent us a heartbeat, and whether the broker is trying to perform a controlled shutdown.

## metadata

## queue

包括EventQueue和KafkaEventQueue