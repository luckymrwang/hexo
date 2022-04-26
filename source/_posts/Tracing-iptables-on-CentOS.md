title: Tracing iptables on CentOS
date: 2022-04-22 23:37:35
tags: [IPTABLES]
---

## 前提条件

- `iptables`已开启并激活

### 步骤 1: 标记要跟踪的数据包

#### DNS端口示例

```js
PORT=53
which sudo || alias sudo='$@'

sudo iptables -t raw -A OUTPUT     -p udp --dport $PORT -j TRACE
sudo iptables -t raw -A OUTPUT     -p tcp --dport $PORT -j TRACE
sudo iptables -t raw -A PREROUTING -p udp --dport $PORT -j TRACE
sudo iptables -t raw -A PREROUTING -p tcp --dport $PORT -j TRACE
```

<!-- more -->
### 目标示例

```js
DEST=10.44.0.47
which sudo || alias sudo='$@'

sudo iptables -t raw -A OUTPUT     -d $DEST -j TRACE
sudo iptables -t raw -A PREROUTING -d $DEST -j TRACE
```

### 步骤 2 (可选): 查看`iptables`跟踪配置

```js
sudo iptables -t raw -L PREROUTING --line-numbers

# output (DNS example):
Chain PREROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    TRACE      udp  --  anywhere             anywhere             udp dpt:domain
2    TRACE      tcp  --  anywhere             anywhere             tcp dpt:domain
```

### 步骤 3: 激活跟踪

```js
modprobe nf_log_ipv4
sudo sysctl net.netfilter.nf_log.2=nf_log_ipv4

# output:
net.netfilter.nf_log.2 = nf_log_ipv4
```

### 步骤 4: 查看跟踪日志

未过滤

```js
dmesg | grep TRACE
```

或者查看 `messages`:

```js
sudo tail -f /var/log/messages | grep TRACE

# output (DNS example on a kubernets system nslookup from a container to coredns):
Mar 30 19:33:27 dev-node1 kernel: TRACE: raw:PREROUTING:policy:3 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:PREROUTING:rule:1 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:KUBE-SERVICES:rule:13 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:KUBE-MARK-MASQ:rule:1 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:KUBE-MARK-MASQ:return:2 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x4000
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:KUBE-SERVICES:rule:14 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x4000
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:KUBE-SVC-TCOU7JCQXEZGVUNU:rule:2 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x4000
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:KUBE-SEP-VQ37SWWSIRRGCSAM:rule:2 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x4000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:FORWARD:rule:1 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x4000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS:rule:5 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x4000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-DEFAULT:rule:35 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x4000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-ACCEPT:rule:1 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x4000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-ACCEPT:return:2 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-DEFAULT:rule:36 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS:return:9 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:FORWARD:rule:2 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:WEAVE-NPC:rule:4 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: filter:WEAVE-NPC-DEFAULT:rule:21 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:POSTROUTING:rule:1 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:CNI-HOSTPORT-MASQ:return:2 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:POSTROUTING:rule:2 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:27 dev-node1 kernel: TRACE: nat:KUBE-POSTROUTING:rule:1 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b SRC=10.44.0.44 DST=10.44.0.47 LEN=90 TOS=0x00 PREC=0x00 TTL=64 ID=20539 PROTO=UDP SPT=43834 DPT=53 LEN=70 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: raw:PREROUTING:policy:3 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:PREROUTING:rule:1 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-SERVICES:rule:13 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-MARK-MASQ:rule:1 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-MARK-MASQ:return:2 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-SERVICES:rule:14 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-SVC-TCOU7JCQXEZGVUNU:rule:1 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-SEP-LRVEW52VMYCOUSMZ:rule:2 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:FORWARD:rule:1 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS:rule:5 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-DEFAULT:rule:35 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-ACCEPT:rule:1 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-ACCEPT:return:2 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-DEFAULT:rule:36 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS:return:9 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:FORWARD:rule:2 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC:rule:3 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge MAC=aa:c8:81:ae:ca:48:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:POSTROUTING:rule:1 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:CNI-HOSTPORT-MASQ:return:2 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:POSTROUTING:rule:2 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-POSTROUTING:rule:1 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwe-bridge SRC=10.44.0.44 DST=10.32.0.7 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20790 PROTO=UDP SPT=57769 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: raw:PREROUTING:policy:3 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:PREROUTING:rule:1 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-SERVICES:rule:13 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-MARK-MASQ:rule:1 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-MARK-MASQ:return:2 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-SERVICES:rule:14 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-SVC-TCOU7JCQXEZGVUNU:rule:2 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-SEP-VQ37SWWSIRRGCSAM:rule:2 IN=weave OUT= PHYSIN=vethwepl84ee671 MAC=c6:1c:a7:ba:ed:1e:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.96.0.10 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:FORWARD:rule:1 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS:rule:5 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-DEFAULT:rule:35 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-ACCEPT:rule:1 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x4000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-ACCEPT:return:2 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS-DEFAULT:rule:36 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-EGRESS:return:9 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:FORWARD:rule:2 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC:rule:4 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: filter:WEAVE-NPC-DEFAULT:rule:21 IN=weave OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b MAC=36:3c:8f:4d:5f:c3:fe:fe:b6:71:c4:b7:08:00 SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:POSTROUTING:rule:1 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:CNI-HOSTPORT-MASQ:return:2 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:POSTROUTING:rule:2 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
Mar 30 19:33:28 dev-node1 kernel: TRACE: nat:KUBE-POSTROUTING:rule:1 IN= OUT=weave PHYSIN=vethwepl84ee671 PHYSOUT=vethwepl907d81b SRC=10.44.0.44 DST=10.44.0.47 LEN=82 TOS=0x00 PREC=0x00 TTL=64 ID=20808 PROTO=UDP SPT=53495 DPT=53 LEN=62 MARK=0x44000
```

#### 用端口过滤

```js
PORT=53 tail -f /var/log/messages | grep "DPT=$PORT"
```

#### 按目标过滤

```js
DEST=10.44.0.47 tail -f /var/log/messages | grep "D=$DEST"
```

### 步骤 5: 关闭跟踪

#### 步骤 5.1: 查看规则

```js
sudo iptables -t raw -L PREROUTING --line-numbers
# output (DNS example):
Chain PREROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    TRACE      udp  --  anywhere             anywhere             udp dpt:domain
2    TRACE      tcp  --  anywhere             anywhere             tcp dpt:domain

-----

sudo iptables -t raw -L OUTPUT --line-numbers 
# output (DNS example):
Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination
1    TRACE      udp  --  anywhere             anywhere             udp dpt:domain
2    TRACE      tcp  --  anywhere             anywhere             tcp dpt:domain
```

#### 步骤 5.2: 删除规则


```js
sudo iptables -t raw -D PREROUTING 2
sudo iptables -t raw -D PREROUTING 1
sudo iptables -t raw -D OUTPUT     2
sudo iptables -t raw -D OUTPUT     
```

