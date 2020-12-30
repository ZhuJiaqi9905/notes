物理内存：

虚拟地址：一个程序认为自己占据整个物理空间。

映射需要CPU中的mmu单元。它依赖的单位是Page: 4kB。随用随申请，不会全量分配。会出现缺页，cpu会出现缺页异常。进行中断。

操作系统：

- 代码段
- 数据段
- 栈
- 堆

`pcstat /bin/bash`: 询问这个文件的页加载了多少。

一个程序有很多页，但是并不是都Cache到了。

page cache是内核维护的中间层。page cache使用多大内存？是否淘汰？是否延时，是否丢数据？

`ll -h && pcstat /test`

BufferedIO为什么比普通的快：因为JVM开了8kB数组，8kB满了再调用系统调用。内核态和用户态切换次数不同。

page cache能优化IO性能，但是可能丢失数据。

flush

cache和dirty不一样。可能一个dirty会写到磁盘，然后它依然cache着。