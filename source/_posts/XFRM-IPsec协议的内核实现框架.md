title: XFRM -- IPsec协议的内核实现框架
date: 2022-05-23 14:16:10
tags: [Linux]
---

`IPsec`协议帮助IP层建立安全可信的数据包传输通道。当前已经有了如[StrongSwan](https://link.segmentfault.com/?enc=yiJRfjDaOBtxIPM5EjpbsA%3D%3D.Ex4qizN1gKKgbufiqSFjTmO3Q2gj2DujT2N7t2kQhKM%3D)、[OpenSwan](https://link.segmentfault.com/?enc=T3gYD1nBaO8i7odY0GxMxQ%3D%3D.%2Bn11tAQvYuD0p2voWWWbrQiAh8iqTkWiqQfS536cp0M%3D)等比较成熟的解决方案，而它们都使用了Linux内核中的`XFRM`框架进行报文接收发送。

![image](/images/xfrm1.webp)

![image](/images/xfrm2.gif)

<!-- more -->
`XFRM`的正确读音是`transform`(转换), 这表示内核协议栈收到的`IPsec`报文需要经过转换才能还原为原始报文；同样地，要发送的原始报文也需要转换为IPsec报文才能发送出去。

## Overview

### XFRM 实例

IPsec中有两个重要概念：安全关联(Security Association)和安全策略(Security Policy)，这两类信息都需要存放在内核XFRM。核XFRM使用`netns_xfrm`这个结构来组织这些信息，它也被称为xfrm instance(实例)。从它的名字也可以看出来，这个实例是与network namespace相关的，每个命名空间都有这样的一个实例，实例间彼此独立。所以同一台主机上的不同容器可以互不干扰地使用XFRM

```c
struct net
{
    ......
   #ifdef CONFIG_XFRM
    struct netns_xfrm    xfrm;
    #endif 
    ......
}
```

### Netlink 通道

上面提到了Security Association和Security Policy信息，这些信息一般是由用户态IPsec进程(eg. StrongSwan)下发到内核XFRM的，这个下发的通道在network namespace初始化时创建。

```c
static int __net_init xfrm_user_net_init(struct net *net)
{
    struct sock *nlsk;
    struct netlink_kernel_cfg cfg = {
        .groups    = XFRMNLGRP_MAX,
        .input    = xfrm_netlink_rcv,
    };

    nlsk = netlink_kernel_create(net, NETLINK_XFRM, &cfg);
    ......
    return 0;
}
```

这样，当用户下发IPsec配置时，内核便可以调用 `xfrm_netlink_rcv()` 来接收

### XFRM State

`XFRM`使用`xfrm_state`示IPsec协议栈中的Security Association，它表示了一条单方向的IPsec流量所需的一切信息，包括模式(Transport或Tunnel)、密钥、replay参数等信息。用户态IPsec进程通过发送一个XFRM_MSG_NEWSA请求，可以让`XFRM`创建一个`xfrm_state`结构

`xfrm_state`包含的字段很多，这里就不贴了，仅仅列出其中最重要的字段：

- id: 它是一个`xfrm_id`结构，包含该SA的目的地址、SPI、和协议(AH/ESP)
- props：表示该SA的其他属性，包括IPsec Mode(Transport/Tunnel)、源地址等信息

每个`xfrm_state`在内核中会加入多个哈希表，因此，内核可以从多个特征查找到同一个个SA：

- xfrm_state_lookup()： 通过指定的SPI信息查找SA
- xfrm_state_lookup_byaddr(): 通过源地址查找SA
- xfrm_state_find(): 通过目的地址查找SA

用户可以通过`ip xfrm state ls`命令列出当前主机上的`xfrm_state`

```bash
src 192.168.0.1 dst 192.168.0.2
    proto esp spi 0xc420a5ed(3290473965) reqid 1(0x00000001) mode tunnel
    replay-window 0 seq 0x00000000 flag af-unspec (0x00100000)
    auth-trunc hmac(sha256) 0xa65e95de83369bd9f3be3afafc5c363ea5e5e3e12c3017837a7b9dd40fe1901f (256 bits) 128
    enc cbc(aes) 0x61cd9e16bb8c1d9757852ce1ff46791f (128 bits)
    anti-replay context: seq 0x0, oseq 0x1, bitmap 0x00000000
    lifetime config:
      limit: soft (INF)(bytes), hard (INF)(bytes)
      limit: soft (INF)(packets), hard (INF)(packets)
      expire add: soft 1004(sec), hard 1200(sec)
      expire use: soft 0(sec), hard 0(sec)
    lifetime current:
      84(bytes), 1(packets)
      add 2019-09-02 10:25:39 use 2019-09-02 10:25:39
    stats:
      replay-window 0 replay 0 failed 0
```

### XFRM Policy

`XFRM`使用`xfrm_policy`表示IPsec协议栈中的Security Policy，用户通过下发这样的规则，可以让XFRM允许或者禁止某些特征的流的发送和接收。用户态IPsec进程通过发送一个`XFRM_MSG_POLICY`请求，可以让XFRM创建一个`xfrm_state`结构

```c
struct xfrm_policy {
    ......
    struct hlist_node    bydst;
    struct hlist_node    byidx;

    /* This lock only affects elements except for entry. */
    rwlock_t        lock;
    atomic_t        refcnt;
    struct timer_list    timer;

    struct flow_cache_object flo;
    atomic_t        genid;
    u32            priority;
    u32            index;
    struct xfrm_mark    mark;
    struct xfrm_selector    selector;
    struct xfrm_lifetime_cfg lft;
    struct xfrm_lifetime_cur curlft;
    struct xfrm_policy_walk_entry walk;
    struct xfrm_policy_queue polq;
    u8            type;
    u8            action;
    u8            flags;
    u8            xfrm_nr;
    u16            family;
    struct xfrm_sec_ctx    *security;
    struct xfrm_tmpl           xfrm_vec[XFRM_MAX_DEPTH];
    struct rcu_head        rcu;
};
```

这个结构的字段很多，但大部分并不用关心，我们重点关注下面列举出的这几个字段就行：

- selector：表示该Policy匹配的流的特征
- action：取值为XFRM_POLICY_ALLOW(0)或XFRM_POLICY_BLOCK(1)，前者表示允许该流量，后者表示不允许。
- xfrm_nr: 表示与这条Policy关联的template的数量，template可以理解为`xfrm_state`的简化版本，xfrm_nr决定了流量进行转换的次数，通常这个值为1
- xfrm_vec: 表示与这条Policy关联的template，数组的每个元素是`xfrm_tmpl`, 一个`xfrm_tmpl`可以还原(resolve)成一个完成`state`

与`xfrm_state`类似，用户可以通过`ip xfrm policy ls`命令列出当前主机上的`xfrm_policy`

```bash
src 10.1.0.0/16 dst 10.2.0.0/16 uid 0
    dir out action allow index 5025 priority 383615 ptype main share any flag  (0x00000000)
    lifetime config:
      limit: soft (INF)(bytes), hard (INF)(bytes)
      limit: soft (INF)(packets), hard (INF)(packets)
      expire add: soft 0(sec), hard 0(sec)
      expire use: soft 0(sec), hard 0(sec)
    lifetime current:
      0(bytes), 0(packets)
      add 2019-09-02 10:25:39 use 2019-09-02 10:25:39
    tmpl src 192.168.0.1 dst 192.168.0.2
        proto esp spi 0xc420a5ed(3290473965) reqid 1(0x00000001) mode tunnel
        level required share any 
        enc-mask ffffffff auth-mask ffffffff comp-mask ffffffff
```

## 接收发送IPsec报文

### 接收

下图展示了`XFRM`框架接收IPsec报文的流程：

![image](/images/xfrm3.webp)

从整体上看，IPsec报文的接收是一个迂回的过程，IP层接收时，根据报文的protocol字段，如果它是IPsec类型(AH、ESP)，则会进入`XFRM`框架进行接收，在此过程里，比较重要的过程是`xfrm_state_lookup()`, 该函数查找SA，如果找到之后，再根据不同的协议和模式进入不同的处理过程，最终，将原始报文的信息获取出来，重入`ip_local_deliver()`.然后，还需经历XFRM Policy的过滤，最后再上送到应用层。

### 发送

下图展示了`XFRM`框架发送IPsec报文的流程：

![image](/images/xfrm4.png)

`XFRM`在报文路由查找后查找是否有满足条件的SA，如果没有，则直接走`ip_output()`,否则进入`XFRM`的处理过程，根据模式和协议做相应处理，最后殊途同归到`ip_output()`

本文引自[这里](https://segmentfault.com/a/1190000020412259)



