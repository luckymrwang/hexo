title: 探究K8S Service内部iptables路由规则
date: 2021-02-20 10:37:20
tags: [Kubernetes]
---

### 前言

在K8S集群内部，应用常使用Service互访，那么，了解Service技术优缺点将有利于应用规划与部署，鉴于此，本文将通过简单案例以探索Cluster-Ip类型Service服务的利弊。

为便于讲解，我们先创建如下应用及Service服务：

```bash
# kubectl run --image=nginx nginx-web-1 --image-pull-policy='IfNotPresent'
# kubectl expose deployment nginx-web-1 --port=80 --target-port=80
```
<!-- more -->
### Service探索

作者的K8S环境是1.9版本，其Service内部服务由Kube-Proxy1提供，且默认用iptables技术实现，故本文探索K8S集群Service技术，即研究iptables在K8S上的技术实现。

### Service Route（服务路由）

如下可知，通过nginx-web-1服务可实际访问到后端pod：

```bash
# nginx pod ip地址：
# kubectl describe pod nginx-web-1-fb8d45f5f-dcbtt | grep "IP"
IP:             10.129.1.22

# Service服务，通过172.30.132.253:80则实际访问到10.129.1.22:80
# kubectl describe svc nginx-web-1 
...
Type:              ClusterIP
IP:                172.30.132.253
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.129.1.22:80
Session Affinity:  None
...

# 重置nginx web页面：
# kubectl exec -it nginx-web-1-fb8d45f5f-dcbtt -- \
               sh -c "echo hello>/usr/share/nginx/html/index.html"

# curl 10.129.1.22
hello
# curl 172.30.132.253
hello
```

Service服务分配的CLUSTER-IP以及监听的端口均虚拟的，即在K8S集群节点上执行ip a与netstat -an命令均无法找到，其实际上，IP与Port是由iptables配置在每K8S节点上的。在节点上执行如下命令可找到此Service相关的iptables配置，简析如下：

- 当通过Service服务IP：172.30.132.253:80访问时，匹配第3条规则链（KUBE-SERVICES）后，跳转到第4条子链（KUBE-SVC-...）上；
- 第4条子链做简单注释后，继而跳转到第1、2规则链（KUBE-SEP-...）上；
- 当源Pod通过Service访问自身时，匹配第1条规则，继而跳转到KUBE-MARK-MASQ链中；
- 匹配到第2条规则，此时通过DNAT被重定向到后端Pod：108.29.1.22:80。


```bash
# iptables-save | grep nginx-web-1
-A KUBE-SEP-UWNFTKZFYWNNNTK7 -s 10.129.1.22/32 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-MARK-MASQ
-A KUBE-SEP-UWNFTKZFYWNNNTK7 -p tcp -m comment --comment "demo/nginx-web-1:" \
   -m tcp -j DNAT --to-destination 10.129.1.22:80
-A KUBE-SERVICES -d 172.30.132.253/32 -p tcp -m comment \
   --comment "demo/nginx-web-1: cluster IP" -m tcp --dport 80 -j KUBE-SVC-SNP24T7IBBNZDJ76
-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-SEP-UWNFTKZFYWNNNTK7
```

详细分析iptables规则，执行iptables-save命令可发现nat的PREROUTING与OUTPUT链中均有KUBE-SERVICES规则链，且处于第一顺位。

```bash
*nat
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

当通过Service访问应用时，流量经由nat表中的PREROUTING规则链处理后，跳转到KUBE-SERVICES子链，而此链包含了对具体Service处理的规则。如下所示，访问172.30.132.253:80将被跳转到KUBE-SEP-...子规则链中。

```bash
-A KUBE-SERVICES -d 172.30.132.253/32 -p tcp -m comment \
   --comment "demo/nginx-web-1: cluster IP" -m tcp --dport 80 -j KUBE-SVC-SNP24T7IBBNZDJ76
-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-SEP-UWNFTKZFYWNNNTK7
```

### Loadbalance（负载均衡）

执行如下命令将Deployment扩展为3个Pod后，继而再观察Service负载均衡方面的技术或问题。

```bash
# kubectl scale deploy/nginx-web-1 --replicas=3
```

再次dump防火墙规则，发现Service经由iptables的statistic模块，以random方式均衡的分发流量，也即负载均衡模式为轮训。

- 存在3条DNAT与KUBE-MARK-MASQ规则，分别对应3个后端Pod实地址；
- KUBE-SERVICES链中存在3条子链，除最后一条KUBE-SVC-...子链外，其余子链使用模块statistic的random模式做流量分割或负载均衡：第1条KUBE-SVC-...应用33%流量，第2条KUBE-SVC-...规则应用剩余的50%流量，第3条KUBE-SVC-...规则应用最后的流量。

```bash
# iptables-save | grep nginx-web-1
-A KUBE-SEP-BI762VOIAZZWU5S7 -s 10.129.1.27/32 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-MARK-MASQ
-A KUBE-SEP-BI762VOIAZZWU5S7 -p tcp -m comment --comment "demo/nginx-web-1:" \
   -m tcp -j DNAT --to-destination 10.129.1.27:80

-A KUBE-SEP-CDQIKEVSTA766BRK -s 10.129.1.28/32 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-MARK-MASQ
-A KUBE-SEP-CDQIKEVSTA766BRK -p tcp -m comment --comment "demo/nginx-web-1:" \
   -m tcp -j DNAT --to-destination 10.129.1.28:80

-A KUBE-SEP-W5HTO42ZVNHJQWBG -s 10.129.3.57/32 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-MARK-MASQ
-A KUBE-SEP-W5HTO42ZVNHJQWBG -p tcp -m comment --comment "demo/nginx-web-1:" \
   -m tcp -j DNAT --to-destination 10.129.3.57:80

-A KUBE-SERVICES -d 172.30.132.253/32 -p tcp -m comment \
   --comment "demo/nginx-web-1: cluster IP" -m tcp --dport 80 -j KUBE-SVC-SNP24T7IBBNZDJ76

-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-BI762VOIAZZWU5S7
-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-CDQIKEVSTA766BRK
-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-SEP-W5HTO42ZVNHJQWBG
```

### Session Affinity（会话保持）

如下所示，调整Service服务，打开会话保持功能，并设置会话保持期限为3小时（PS：若不设置，则默认是3小时）：

```bash
# kubectl edit svc nginx-web-1
...
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
...
```

继续观察iptables实现，发现在原有基础上，iptables规则中添加了recent模块，此模块被用于会话保持功能，故kube-proxy通过在iptables中结合statistic与recent模块，实现了Service的轮训负载均衡与会话保持功能。

- 通过Service服务访问应用，封包进入KUBE-SERVICES规则链，并跳转到KUBE-SVC-...子链中；
- 在KUBE-SVC-SNP...子链中，recent位于statistic模块前，故而，有如下情况出现：

	- 当客户端第一次访问Service时，KUBE-SVC-...子链中的规则（-m recent --rcheck --seconds 10800 --reap ...--rsource）池中未记录客户端地址，故封包匹配失败，从而封包被后续的statistic模块规则处理后，均衡分发到KUBE-SEP-...子链中，此链中使用recent模块的--set参数将客户源地址记录到规则池后，DNAT到实际后端实例上；
	- KUBE-SVC-...子链中recent模块配置了源地址记录期限，若客户端3（--seconds 10800 --reap）小时内未访问服务，则recent规则池中的客户端记录将被移除，此时客户端再次访问Service就如同第一次访问Service一样；
	- 当客户端在3小时内再次访问Service时，匹配KUBE-SVC-...子链中的recent模块规则后，跳转到KUBE-SEP子链，其规则中recent模块--set参数将更新规则池中的Record TTL，而后DNAT到实际后端实例上；


```bash
# iptables-save | grep nginx-web-1
-A KUBE-SEP-BI762VOIAZZWU5S7 -s 10.129.1.27/32 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-MARK-MASQ
-A KUBE-SEP-BI762VOIAZZWU5S7 -p tcp -m comment --comment "demo/nginx-web-1:" \
   -m recent --set --name KUBE-SEP-BI762VOIAZZWU5S7 --mask 255.255.255.255 \
   --rsource -m tcp -j DNAT --to-destination 10.129.1.27:80
# 省略2条类似的KUBE-SEP规则
...

-A KUBE-SERVICES -d 172.30.132.253/32 -p tcp -m comment \
   --comment "demo/nginx-web-1: cluster IP" -m tcp --dport 80 -j KUBE-SVC-SNP24T7IBBNZDJ76

-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-BI762VOIAZZWU5S7 \
   --mask 255.255.255.255 --rsource -j KUBE-SEP-BI762VOIAZZWU5S7
# 省略2条类似的KUBE-SVC规则
...

-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-BI762VOIAZZWU5S7
-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-CDQIKEVSTA766BRK
-A KUBE-SVC-SNP24T7IBBNZDJ76 -m comment --comment "demo/nginx-web-1:" \
   -j KUBE-SEP-W5HTO42ZVNHJQWBG
```

### 总结

K8S中的Service服务可提供负载均衡及会话保持功能，其通过Linux内核netfilter模块来配置iptables实现，网络封包在内核中流转，且规则匹配很少，故效率非常高；而Service负载均衡分发比较薄弱，其通过statistic的random规则实现轮训分发，无法实现复杂的如最小链接分发方式，鉴于此，K8S 1.9后续版本调整了kube-proxy服务，其可通过ipvs实现Service负载均衡功能。



​ 