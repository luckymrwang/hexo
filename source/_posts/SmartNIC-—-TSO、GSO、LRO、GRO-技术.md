title: SmartNIC — TSO、GSO、LRO、GRO 技术
date: 2022-07-27 19:54:08
tags: [Linux]
---
### TSO(TCP Segmentation Offload)

TSO(TCP Segmentation Offload): 是一种利用网卡对大数据包进行分片，从而减小CPU负荷的一种技术。

- TSO off 和 GSO off：

![images](/images/tso1.png)

<!-- more -->
- TSO on：

![images](/images/tso2.png)

### GSO(Generic Segmentation Offload)

GSO(Generic Segmentation Offload): 是一种延缓分片技术。它比 TSO 更通用，原因在于它不需要硬件的支持就可以进行分片。

其过程是：首先查询网卡是否支持 TSO 功能，如果硬件支持 TSO 则使用网卡的硬件分片能力执行分片；如果网卡不支持 TSO 功能，则将分片的执行延缓到了将数据推送到网卡的前一刻执行。

- TSO off 和 GSO on：一个大的网络包直到进入网卡前的最后一步才进行分片。

![images](/images/tso3.png)

### LRO(Large Receive Offload)

LRO(Large Receive Offload)：将网卡接收到的多个数据包合并成一个大的数据包，然后再传递给网络协议栈处理的技术。这样提高系统接收数据包的能力，减轻CPU负载。

- LRO off 和 GRO off：

![images](/images/tso4.png)

- LRO on：数据一进入网卡立刻进行了合并。

![images](/images/tso5.png)

### GRO(Generic Receive Offload)

GRO(Generic Receive Offload)：是 LRO 的软实现，GRO 的合并条件更加的严格和灵活。

- GRO on：

![images](/images/tso6.png)

### 关联

- [linux内核协议栈 TCP层数据发送之TSO/GSO](https://blog.csdn.net/wangquan1992/article/details/109018488)


