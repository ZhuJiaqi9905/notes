# DPDK

kernel只有简单的驱动。

rte_eal是一个环境抽象层。

`ret_eal_init()`

具体收发包。

descriptor<-> mbuf

当一个包过来，网卡拿一个descriptor，然后do DMA。

`rte_eth_rx_burst`



混杂模式：类似于广播





成功发送的包，dpdk会释放掉。不成功的包需要手动释放掉。

应用层 mbuffer dpdk ring buffer NIC

内存池里有很多mbuf结构

ring buffer里面存着descriptor，每个指向一个mbuf。

mem poll的大小要比ring buffer大一些。



bluefield NIC是什么？