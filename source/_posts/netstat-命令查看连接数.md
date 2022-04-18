title: netstat 命令查看连接数
date: 2022-03-25 10:29:28
tags: [Linux]
---

>显示所有活动的网络连接

```sh
netstat -na
```

>查看同时连接到哪个服务器 IP 比较多，cc 攻击用。使用双网卡或多网卡可用

```sh
netstat -an|awk  '{print $4}'|sort|uniq -c|sort -nr|head
```

<!-- more -->
>查看哪些 IP 连接到服务器连接多，可以查看连接异常 IP

```sh
netstat -an|awk -F: '{print $2}'|sort|uniq -c|sort -nr|head
```

>显示所有 80 端口的网络连接并排序。这里的 80 端口是 http 端口，所以可以用来监控 web 服务。如果看到同一个 IP 有大量连接的话就可以判定单点流量攻击了

```sh
netstat -an | grep :80 | sort
```

>这个命令可以查找出当前服务器有多少个活动的 SYNC_REC 连接。正常来说这个值很小，最好小于 5。 当有 Dos 攻击或的时候，这个值相当的高。但是有些并发很高的服务器，这个值确实是很高，因此很高并不能说明一定被攻击

```sh
netstat -n -p|grep SYN_REC | wc -l
```

>列出所有连接过的 IP 地址

```sh
netstat -n -p | grep SYN_REC | sort -u
```

>列出所有发送 SYN_REC 连接节点的 IP 地址

```sh
netstat -n -p | grep SYN_REC | awk '{print $5}' | awk -F: '{print $1}'
```

>使用 netstat 命令计算每个主机连接到本机的连接数

```sh
netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
```

>列出所有连接到本机的 UDP 或者 TCP 连接的 IP 数量

```sh
netstat -anp |grep 'tcp|udp' | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
```

>检查 ESTABLISHED 连接并且列出每个 IP 地址的连接数量

```sh
netstat -ntu | grep ESTAB | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr
```

>查看并发请求数及其TCP连接状态

```sh
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

>列出所有连接到本机 80 端口的 IP 地址和其连接数。80 端口一般是用来处理 HTTP 网页请求

```sh
netstat -plan|grep :80|awk {'print $5'}|cut -d: -f 1|sort|uniq -c|sort -nk 1
```

>显示连接 80 端口前 10 的 ip，并显示每个 IP 的连接数。这里的 80 端口是 http 端口，所以可以用来监控 web 服务。如果看到同一个 IP 有大量连接的话就可以判定单点流量攻击了

```sh
netstat -antp | awk '$4 ~ /:80$/ {print $4" "$5}' | awk '{print $2}'|awk -F : {'print $1'} | uniq -c | sort -nr | head -n 10
```