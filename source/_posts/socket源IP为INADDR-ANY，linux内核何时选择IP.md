title: socket源IP为INADDR_ANY，linux内核何时选择IP
date: 2022-09-29 12:24:27
tags: [Linux]
---

内核版本3.10.14为例

## 绑定源IP时，bind操作

`af_inet.c` 中 `inet_bind` 函数

```c
int inet_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
{
	struct sockaddr_in *addr = (struct sockaddr_in *)uaddr;
	struct sock *sk = sock->sk;
	struct inet_sock *inet = inet_sk(sk);
......
inet->inet_rcv_saddr = inet->inet_saddr = addr->sin_addr.s_addr;
......
}
```
<!-- more -->

源IP保存在 `inet_sock` 的参数 `inet_saddr` 中。

## 不绑源IP时

当不绑定源IP或者源IP绑定为 `INADDR_ANY` 时，系统何时选择IP

答：`tcp` 在 `connect` 时，`udp` 在 `send_msg` 时


## TCP

### tcp_v4_connect

`connect` 系统调用对应内核实现 `tcp_v4_connect`

```c
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
	struct sockaddr_in *usin = (struct sockaddr_in *)uaddr;
	struct inet_sock *inet = inet_sk(sk);
	struct tcp_sock *tp = tcp_sk(sk);
	......
	fl4 = &inet->cork.fl.u.ip4;
	/*tcp_v4_connect时会选择路由，并且填充源地址*/
	rt = ip_route_connect(fl4, nexthop, inet->inet_saddr,
			      RT_CONN_FLAGS(sk), sk->sk_bound_dev_if,
			      IPPROTO_TCP,
			      orig_sport, orig_dport, sk, true);

	/*如果源地址为0，则填充源地址*/
	if (!inet->inet_saddr)
		inet->inet_saddr = fl4->saddr;
	inet->inet_rcv_saddr = inet->inet_saddr;
}
```

**查路由，路由返回值中有源IP信息，然后设置到 `inet_sock->inet_saddr` 中。**

## UDP

udp 是非连接套接字，一个 socket 可以和多个对端通信，针对不同的对端选择的源IP可能是不同的；在 `send_msg` 中选择路由和源IP；

### udp_sendmsg

```c
int udp_sendmsg(struct kiocb *iocb, struct sock *sk, struct msghdr *msg,
		size_t len)
{
	struct inet_sock *inet = inet_sk(sk);
	struct udp_sock *up = udp_sk(sk);
	struct flowi4 fl4_stack;
	struct flowi4 *fl4;
......
if (connected)
		rt = (struct rtable *)sk_dst_check(sk, 0);
		
	/*如果是无连接的udp*/
	if (rt == NULL) {
		struct net *net = sock_net(sk);
		fl4 = &fl4_stack;
		flowi4_init_output(fl4, ipc.oif, sk->sk_mark, tos,
				   RT_SCOPE_UNIVERSE, sk->sk_protocol,
				   inet_sk_flowi_flags(sk)|FLOWI_FLAG_CAN_SLEEP,
				   faddr, saddr, dport, inet->inet_sport);

		security_sk_classify_flow(sk, flowi4_to_flowi(fl4));
		rt = ip_route_output_flow(net, fl4, sk);
......
		if (connected)
			sk_dst_set(sk, dst_clone(&rt->dst));
	}
	/*后面更具flowi->saddr构造skb*/
}
```

结论：源IP选择是在查找路由的操作中，查找路由的结构提 `flowi4` 结构体成员 `saddr` 会返回源IP信息。

## 路由查找时如何确定源IP

本地发包查路由核心函数 `__ip_route_output_key`

```c
struct rtable *__ip_route_output_key(struct net *net, struct flowi4 *fl4)
{
......
fib_lookup(net, fl4, &res);
/*如果源IP为空，则设置成返回的源IP*/
if (!fl4->saddr)
		fl4->saddr = FIB_RES_PREFSRC(net, res);

	dev_out = FIB_RES_DEV(res);
	fl4->flowi4_oif = dev_out->ifindex;
}
```

`FIB_RES_PREFSRC` 从路由查找结果 `res` 中选择源IP信息

```c
#define FIB_RES_PREFSRC(net, res)	((res).fi->fib_prefsrc ? : \
					 FIB_RES_SADDR(net, res))

#define FIB_RES_SADDR(net, res)				\
	((FIB_RES_NH(res).nh_saddr_genid ==		\
	  atomic_read(&(net)->ipv4.dev_addr_genid)) ?	\
	 FIB_RES_NH(res).nh_saddr :			\
	 fib_info_update_nh_saddr((net), &FIB_RES_NH(res)))
```

优先级 `prefsrc > fib_nh.nh_saddr`

`prefsrc` 是路由条目的属性，一般子网路由会有该属性，在设置路由条目时可以指定。

`fib_nh` 是路由条目。`nh_saddr` 通过 `fib_info_update_nh_saddr` 赋值

```c
__be32 fib_info_update_nh_saddr(struct net *net, struct fib_nh *nh)
{
	nh->nh_saddr = inet_select_addr(nh->nh_dev,
					nh->nh_gw,
					nh->nh_parent->fib_scope);
	nh->nh_saddr_genid = atomic_read(&net->ipv4.dev_addr_genid);

	return nh->nh_saddr;
}
```

路由条目的 `nh_saddr` 是通过 `inet_select_addr` 选择的，

`inet_select_addr` 函数是找到设备 `nh_dev` 的所有IP中和网关 `nh_gw` 在同一网段的IP；

```c
__be32 inet_select_addr(const struct net_device *dev, __be32 dst, int scope)
{
......
for_primary_ifa(in_dev) {
		if (ifa->ifa_scope > scope)
			continue;
		/*找到跟网关同一网段的IP*/
		if (!dst || inet_ifa_match(dst, ifa)) {
			addr = ifa->ifa_local;
			break;
		}
		if (!addr)
			addr = ifa->ifa_local;
	} endfor_ifa(in_dev);
}
```




