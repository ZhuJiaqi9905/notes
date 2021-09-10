# RDMA 操作

查看IB设备: `ibstat`

![](figs/78.png)

注意1：**双端口的IB网卡被视为两个设备**

注意 2：在RDMA编程的语境下，`ib_port` 与网卡（HCA）的端口不是一个概念。正如上文所说，HCA的一个物理端口对应一个IB设备，`ib_port`是IB设备的子概念，如上图所示，每个CA都有一个port，其id为1。

