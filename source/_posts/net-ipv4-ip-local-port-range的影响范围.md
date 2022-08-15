title: net.ipv4.ip_local_port_range的影响范围
date: 2022-07-16 07:59:42
tags: [Linux]
---

### 前言

网上关于 net.ipv4.ip_local_port_range 的值的效果众说纷纭（下面所说的连接都假定使用的是相同的协议(都是 TCP 或 UDP)）:

- 大部分文章都说这个值决定了客户端的一个 ip 可用的端口数量，即一个 ip 最多只能创建 60K 多一点的连接（1025-65535），如果要突破这个限制需要客户端机器绑定多个 ip。
- 还有部分文章说的是这个值决定的是 socket 四元组中的本地端口数量，即一个 ip 对同一个目标 ip+port 最多可以创建 60K 多一点连接，只要目标 ip 或端口不一样就可以使用相同的本地端口，不一定需要多个客户端 ip 就可以突破端口数量限制。

<!-- more -->
文档中的介绍也很模糊:

```js
ip_local_port_range - 2 INTEGERS
    Defines the local port range that is used by TCP and UDP to
    choose the local port. The first number is the first, the
    second the last local port number.
    If possible, it is better these numbers have different parity.
    (one even and one odd values)
    The default values are 32768 and 60999 respectively.
```

下面就来做一些实验来确认这个选项的实际效果。

实验环境:

```js
$ uname -a
Linux vagrant 4.15.0-29-generic #31-Ubuntu SMP Tue Jul 17 15:39:52 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

### 相同目标 ip 和相同目标端口下的端口数量限制

先设置 ip_local_port_range 的值为非常小的范围:

```js
$ echo "61000 61001" | sudo tee /proc/sys/net/ipv4/ip_local_port_range
61000 61001

$ cat /proc/sys/net/ipv4/ip_local_port_range
61000       61001
```

然后对相同 ip 和端口发送 tcp 请求。创建两个连接，达到最大端口数量限制:

```js
$ nohup nc 123.125.114.144 80 -v &
[1] 16196
$ nohup: ignoring input and appending output to 'nohup.out'

$ nohup nc 123.125.114.144 80 -v &
[2] 16197
$ nohup: ignoring input and appending output to 'nohup.out'

$ ss -ant |grep 10.0.2.15:61
ESTAB   0        0                10.0.2.15:61001       123.125.114.144:80
ESTAB   0        0                10.0.2.15:61000       123.125.114.144:80
```

然后再创建第三个连接，此时预期应该会失败，因为超出的端口数量现在:

```js
vagrant@vagrant:~$ nc 123.125.114.144 80 -v
nc: connect to 123.125.114.144 port 80 (tcp) failed: Cannot assign requested address
```

可以看到确实如预期的失败了。

### 相同目标 ip 不同目标端口

下面看看相同目标 ip 不同目标端口是否可以突破这个端口限制:

```js
$ nohup nc 123.125.114.144 443 -v &
[3] 16215
$ nohup: ignoring input and appending output to 'nohup.out'

$ nohup nc 123.125.114.144 443 -v &
[4] 16216
$ nohup: ignoring input and appending output to 'nohup.out'

$ ss -ant |grep 10.0.2.15:61
ESTAB   0        0                10.0.2.15:61001       123.125.114.144:443
ESTAB   0        0                10.0.2.15:61001       123.125.114.144:80
ESTAB   0        0                10.0.2.15:61000       123.125.114.144:443
ESTAB   0        0                10.0.2.15:61000       123.125.114.144:80
```

可以看到相同目标 ip 不同目标端口下，每个目标端口都有一个独立的端口限制，即，相同源 ip 的源端口是可以相同的。

按照推测这两个目标端口应该只能创建四个连接，下面试试看:

```js
$ ss -ant |grep 10.0.2.15:61
ESTAB   0        0                10.0.2.15:61001       123.125.114.144:443
ESTAB   0        0                10.0.2.15:61001       123.125.114.144:80
ESTAB   0        0                10.0.2.15:61000       123.125.114.144:443
ESTAB   0        0                10.0.2.15:61000       123.125.114.144:80

$ nc 123.125.114.144 443 -v
nc: connect to 123.125.114.144 port 443 (tcp) failed: Cannot assign requested address
```

确实是不能再创建连接了，因为每个目标端口都达到了 ip_local_port_range 的限制。

### 多个目标 ip 相同目标端口

下面看一下多个目标 ip 相同目标端口下的情况:

```js
$ nohup nc 220.181.57.216 80 -v &
[5] 16222
$ nohup: ignoring input and appending output to 'nohup.out'

$ nohup nc 220.181.57.216 80 -v &
[6] 16223
$ nohup: ignoring input and appending output to 'nohup.out'

$ nc 220.181.57.216 80 -v
nc: connect to 220.181.57.216 port 80 (tcp) failed: Cannot assign requested address

$ ss -ant |grep :80
SYN-SENT  0        1               10.0.2.15:61001      220.181.57.216:80
SYN-SENT  0        1               10.0.2.15:61000      220.181.57.216:80
SYN-SENT  0        1               10.0.2.15:61001      123.125.114.144:80
SYN-SENT  0        1               10.0.2.15:61000      123.125.114.144:80
```

可以看到，每个目标 ip 都有独立的 ip_local_port_range 限制。

### 多个目标 ip 不同目标端口

下面看一下多个目标 ip 相同不同端口下的情况，按照前面的经验两个 ip 加两个端口应该只能创建 8 个连接

```js
$ nohup nc 123.125.114.144 80 -v &

$ nohup nc 123.125.114.144 80 -v &

$ nc 123.125.114.144 80 -v
nc: connect to 123.125.114.144 port 80 (tcp) failed: Cannot assign requested address

$ nohup nc 123.125.114.144 443 -v &

$ nohup nc 123.125.114.144 443 -v &

$ nc 123.125.114.144 443 -v
nc: connect to 123.125.114.144 port 443 (tcp) failed: Cannot assign requested address

$ nohup nc 220.181.57.216 80 -v &

$ nohup nc 220.181.57.216 80 -v &

$ nc 220.181.57.216 80 -v
nc: connect to 220.181.57.216 port 80 (tcp) failed: Cannot assign requested address

$ nohup nc 220.181.57.216 443 -v &

$ nohup nc 220.181.57.216 443 -v &

$ nc 220.181.57.216 443 -v
nc: connect to 220.181.57.216 port 443 (tcp) failed: Cannot assign requested address

$ ss -ant |grep 10.0.2.15:61
SYN-SENT  0        1               10.0.2.15:61001      220.181.57.216:80
ESTAB     0        0               10.0.2.15:61001      123.125.114.144:443
ESTAB     0        0               10.0.2.15:61000      220.181.57.216:443
SYN-SENT  0        1               10.0.2.15:61000      220.181.57.216:80
SYN-SENT  0        1               10.0.2.15:61001      123.125.114.144:80
ESTAB     0        0               10.0.2.15:61000      123.125.114.144:443
SYN-SENT  0        1               10.0.2.15:61000      123.125.114.144:80
ESTAB     0        0               10.0.2.15:61001      220.181.57.216:443
```

可以看到确实如预期的只能创建8个连接。

### 总结
那么是否就可以说前言中的第一种说法就是错的呢，查了一下资料其实也不能说第一种说法是错误的：

- 当系统的内核版本小于 3.2 时，第一种说法是正确的
- 当系统的内核版本大于等于 3.2 时，第二种说法是正确的

### 参考资料

- [kernel.org/doc/Documentation/networking/ip-sysctl.txt](kernel.org/doc/Documentation/networking/ip-sysctl.txt)
- [Tuning your Linux kernel and HAProxy instance for high loads](Tuning your Linux kernel and HAProxy instance for high loads)
- [Load Testing HAProxy (Part 2) – freeCodeCamp.org](Load Testing HAProxy (Part 2) – freeCodeCamp.org)
- [What is the theoretical maximum number of open TCP connections that a modern Linux box can have - Stack Overflow](What is the theoretical maximum number of open TCP connections that a modern Linux box can have - Stack Overflow)
- [**Coping with the TCP TIME-WAIT state on busy Linux servers | Vincent Bernat**](Coping with the TCP TIME-WAIT state on busy Linux servers | Vincent Bernat)
- ['Re: Fix for rare EADDRNOTAVAIL error' - MARC]('Re: Fix for rare EADDRNOTAVAIL error' - MARC)

本文引自[这里](https://mozillazg.com/2019/05/linux-what-net.ipv4.ip_local_port_range-effect-or-mean.html)
