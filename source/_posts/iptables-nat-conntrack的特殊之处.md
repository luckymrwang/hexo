title: iptables nat&conntrack的特殊之处
date: 2022-04-24 10:07:09
tags: [Linux]
---

### 问题与解释

我在`PREROUTING`上做了一个`REDIRECT`端口改写，相应的服务处理请求后会返回应答。

按照我以往的认识，认为回包的流量应该先后经过`OUTPUT`和`POSTROUTING`，所以我利用`iptables -t nat -nvL`去查看`NAT`表在`OUTPUT`链和`POSTROUTING`链上的`packge`计数器，结果发现没有上涨，这让我陷入了沉思。

经过谷歌后找到了完美的解释：[linux-netfilter-how-does-connection-tracking-track-connections-changed-by-nat](https://superuser.com/questions/1269859/linux-netfilter-how-does-connection-tracking-track-connections-changed-by-nat)。

<!-- more -->
实际上我是知道`OUTPUT`链会先过`conntrack`表恢复原始IP关系的，但是超出我理解的是`NAT`表压根就不会再执行。

上述URL中给出了解释：`NAT`表只在连接状态是`NEW`的时候（也就是TCP的第一个握手包）才会执行计算，一旦改写关系存入了`conntrack`，那么这条连接后续的通讯就不会再过`POSTROUTING`和`OUTPUT`上面的`NAT`表了，而是直接换成了匹配`conntrack`来复原连接之前的改写状态。

因此，如果我们想看到回包的package计数器增长，就应该去看`OUTPUT`或者`POSTROUTING`上面的`filter`表计数，一定会看到上涨。

### 再次梳理流程

如果我们是服务端，那么SYN包到达的时候，在`PREROUTING`链的`NAT`表执行过之后（可能做`DNAT`或者`REDIRECT`），路由表将决定是`FORWARD`还是`INPUT`：

- 如果`INPUT`，那么`conntrack`记录就此生成（原理如下），当回包的时候会首先根据`conntrack`作地址复原，并且是不会经过`OUTPUT/POSTROUTING`链`NAT`表（但是会经过`filter`表）的。
- 如果`FORWARD`，那么`conntrack`记录不会立即生成，需要经过`POSTROUTING`之后才知道是否做了`SNAT/MASQUERADE`，此时才会生成`conntrack`记录（原理如下）。当收到上游回包的时候，不会过`PREROUTING`的`NAT`表，而是直接根据`conntrack`复原为原始IP地址，然后直接`FORWARD->POSTROUTING`（不会过`NAT`表）送回原始客户端。

具体原理如下：

![](images/netfilter-conntrack.png)

如上图所示，Netfilter 在四个 Hook 点对包进行跟踪：

- `PRE_ROUTING` 和 `LOCAL_OUT`：调用 `nf_conntrack_in()` 开始连接跟踪， 正常情况下会创建一条新连接记录，然后将 `conntrack entry` 放到 `unconfirmed list`。

	为什么是这两个 `hook` 点呢？因为它们都是新连接的第一个包最先达到的地方，

	- 	`PRE_ROUTING` 是外部主动和本机建连时包最先到达的地方
	- 	`LOCAL_OUT` 是本机主动和外部建连时包最先到达的地方

- `POST_ROUTING` 和 `LOCAL_IN`：调用 `nf_conntrack_confirm()` 将 `nf_conntrack_in()` 创建的连接移到 `confirmed list`。

	同样要问，为什么在这两个 `hook` 点呢？因为如果新连接的第一个包没有被丢弃，那这 是它们离开 `netfilter` 之前的最后 `hook` 点：

	- 外部主动和本机建连的包，如果在中间处理中没有被丢弃，`LOCAL_IN` 是其被送到应用（例如 nginx 服务）之前的最后 hook 点
	- 本机主动和外部建连的包，如果在中间处理中没有被丢弃，`POST_ROUTING` 是其离开主机时的最后 hook 点

### 参考

- [连接跟踪（conntrack）：原理、应用及 Linux 内核实现](https://arthurchiao.art/blog/conntrack-design-and-implementation-zh/#2-netfilter-hook-%E6%9C%BA%E5%88%B6%E5%AE%9E%E7%8E%B0)
- [云计算底层技术-netfilter框架研究](https://opengers.github.io/openstack/openstack-base-netfilter-framework-overview/)