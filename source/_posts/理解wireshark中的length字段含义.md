title: 理解wireshark中的length字段含义
date: 2022-07-25 22:33:33
tags: [wireshark]
---

主要介绍wireshark中的length计算方法，主要涉及MTU、MSS、常见协议头大小。

### length含义

Wireshark is showing you the length of the Ethernet frame that is handed to it, which may or may not include the FCS.

wireshark显示的长度为以太网帧的长度，不包括FCS(Frame check sequence)[帧校验序列]

<!-- more -->

### 常见协议头大小

```js
Ethernet header: 14 bytes # 数据链路层帧头大小，一般为14bytes,其为源MAC(6bytes/48bits)+目的MAC(6bytes/48bits)+类型/长度(2bytes)
IP header (standard): 20 bytes # IP头部大小
ICMP header: 8 bytes     # ICMP头部大小
ICMP payload: 32 bytes   # ICMP缓冲区大小，可以通过ping的-l选项指定大小，默认为32。
FCS 4 bytes of Ethernet checksum # 帧尾CRC校验
```

### length计算

MTU=MSS+IP header(20 bytes)+tcp header(20 bytes)

length=MTU+Ethernet header(14bytes)

其中MSS为Maximum Segment Size，即最大报文段长度，其受MTU大小影响，这里的MTU指的是三层的，二层的MTU固定为1500，不能修改。

MTU为Maximum Transmission Unit,即最大传输单元，需要注意MTU如果太小会影响收到数据包的速度，表现为下载过慢。

```js
三层收到大数据包时，要将数据包分片后再往下层传输，这就是IP分片原理。既然IP分片可以改变一个IP数据包的大小 ，那么IP分片怎么设置呢？
这也是网上人们所谓的修改MTU值达到最佳网速的方法，而这里所说的修改“MTU”大小其实是IP分片的大小。
```

![image](/images/wireshark1.png)

从截图中可以看到MSS为1460，MTU计算后为1500，1460 + 20(IP header) + 20(tcp header) = 1500

### 遇到的问题

- 问题

同一局域网中，同样的操作系统，同样的URL链接，同样的端口，使用ESXI虚拟出来的两台虚机，分别绑定host，使用wget测试下载速度，一个下载200K/s，一个10M/s。

![images](/images/wireshark2.png)

- 解决

抓包后发现：

28.41的Length为1294，MSS为1240,推断其MTU为1280

28.210的Length为1514，MSS为1460，推断其MTU为1500

猜测其网卡设置不同，进入宿主机看网卡设置，28.41的网卡类型不是E1000，28.210的E1000，将28.41的网卡类型改为E1000后测试，恢复。

![images](/images/wireshark3.png)

![images](/images/wireshark4.png)

![images](/images/wireshark5.png)

本文引自[这里](https://www.louxiaohui.com/2018/06/29/understanding-the-length-field-in-wireshark/)