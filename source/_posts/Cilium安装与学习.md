title: Cilium安装与学习
date: 2022-04-20 19:29:27
tags: [Cilium]
---

## 基础概念

#### BPF(CBPF) [说明文档](https://docs.cilium.io/en/latest/bpf/)

BPF 的全称是 Berkeley Packet Filter, 即伯克利报文过滤器，它的设计思想来源于 1992 年的一篇论文“The BSD packet filter: A New architecture for user-level packet capture” （《BSD数据包过滤器：一种用于用户级数据包捕获的新体系结构》）。最初，BPF是在 BSD 内核实现的， 后来，由于其出色的设计思想，其他操作系统也将其引入, 包括 Linux。`tcpdump` 使用的libpcap是基于BPF的，在使用tcpdump或者libpcap时传入的“host 192.168.1.1”、“tcp and port 80”等是过滤表达式。

<!-- more -->
#### EBPF

eBPF 是 extended BPF 的简称,Linux Kernel 3.15 中引入的全新设计, 是对既有BPF架构进行了全面扩展，一方面，支持了更多领域的应用，比如：内核追踪(Kernel Tracing)、应用性能调优/监控、流控(Traffic Control)等；另一方面，在接口的设计以及易用性上，也有了较大的改进。现在ebpf学习资源网站，主要由Cilium团队维护，上面会及时更新BPF技术的文档和视频。

#### VXLAN

全称 Virtual eXtensible Local Area Network，是用三层协议封装二层协议，允许第2层数据包在第3层网络上传输。通过Overlay技术旨在扩展VLAN，以解决大型云计算部署中虚拟网络数量不足的问题。

#### Direct Server Return (DSR)

默认情况下，Cilium的BPF NodePort实现以SNAT模式运行。也就是说，当节点外部流量到达并且节点确定NodePort或ExternalIPs服务的后端位于远程节点时，则该节点将通过执行SNAT代表其将请求重定向到远程后端。这不需要任何额外的MTU更改，而代价是来自后端的答复，即在将数据包直接返回到外部客户端之前，需要在该节点上进行额外的跳回以便在该节点上执行反向SNAT转换。
可以通过将 `global.nodePort.mode` helm选项更改为 `dsr` 来更改此设置，以使Cilium的BPF NodePort实现在DSR模式下运行。在这种模式下，后端直接向外部客户端进行回复，而无需花费额外的跳数，这意味着后端通过使用服务IP /端口作为源进行回复。 DSR当前需要  [Native-Routing](https://docs.cilium.io/en/v1.11/concepts/networking/routing/#arch-direct-routing)，即它不能在任何tunneling模式下工作。
DSR模式的另一个优点是保留了客户端的源IP，因此策略可以在后端节点上对其进行匹配。在SNAT模式下，这是不可能的。给定一个特定的后端可以由多个服务使用，则需要使后端知道它们需要回复的服务IP /端口。因此，Cilium将此信息编码为IPv4选项或IPv6扩展头，但其代价是宣传较低的MTU。对于TCP服务，Cilium仅对SYN数据包的服务IP /端口进行编码。

#### XDP

XDP的意思是express Data Path，它能够在网络包进入用户态直接对网络包进行过滤或者处理。XDP依赖eBPF技术

#### eBPF Host-Routing

即使 Cilium 使用 eBPF 执行网络路由，默认情况下网络数据包仍会遍历节点的常规网络堆栈的某些部分。 这样可以确保所有数据包仍然遍历所有 iptables hooks，以防你依赖它们。然而，这样会增加大量开销。
在 Cilium 1.9 中引入了基于 eBPF 的​​ [​host-routing​​](https://cilium.io/blog/2020/11/10/cilium-19#veth)，来完全绕过 iptables 和上层主机堆栈，与常规 veth 设备操作相比，实现了更快的网络命名空间切换。 如果您的内核支持，此选项会自动启用。

![](/images/cilium01.png)

左侧是标准的网络流向过程，可以发现iptables 会对包进行处理
右侧是启用ebpf host-routing, 直接跨过iptables。
必要条件：

- Kernel >= 5.10
- Direct-routing or tunneling
- eBPF-based 替换kube-proxy
- eBPF-based masquerading

## 组网模式

[官方文档](https://docs.cilium.io/en/v1.11/concepts/networking/routing/)

#### Encapsulation(封装)

默认模式，Cilium 会自动在此模式下运行，因为这种模式对底层网络基础设施的要求最低。
在这种模式下，所有集群节点使用基于 UDP 的封装协议 VXLAN 或 Geneve 形成一个tunnel。 Cilium 节点之间的所有流量都被封装。

##### 网络要求

- 依赖于节点到节点的连接正常。 这意味着如果 Cilium 节点已经可以网络互通。
- 底层网络和防火墙必须允许以下端口：

Encapsulation Mode  | Port Range | Protocol
------------- | ------------ | -------------
VXLAN (Default)  | 8472 | UDP
Geneve   | 6081 | UDP

##### 优势
- 简单

集群节点的网络连接不需要知道 PodCIDR。 集群节点可以产生多个路由或链路层域。 只要集群节点可以使用 IP/UDP 相互访问，底层网络的拓扑就无关紧要。

- 寻址空间

由于不依赖于任何底层网络限制，如果相应地配置了 PodCIDR 大小，可用的寻址空间可能会更大，并且允许每个节点运行任意数量的 Pod。

- 自动配置

当与 编排系统（Kubernetes 等）一起运行时，集群中所有节点的列表（包括它们关联的分配前缀节点）会自动提供给每个cilium-agent。 加入集群的新节点将自动合并到网格中。

- Identity context

协议允许将元数据与网络数据包一起携带。 Cilium 利用这种能力来传输元数据，例如源安全身份。 身份传输是一种优化，旨在避免在远程节点上进行一次身份查找。

##### 缺点
- MTU Overhead

由于添加了封装header， MTU过大低于本地路由（VXLAN 的每个网络数据包 50 字节）。 这会导致特定网络连接的最大吞吐率较低。 可以通过启用巨型帧（每 1500 字节 50 字节的开销与每 9000 字节的 50 字节开销）在很大程度上得到缓解。

### Native-Routing

1. 此模式下，Cilium 会将所有未发送到另一个本地端点的数据包委托给 Linux 内核的路由子系统。 这意味着数据包将被路由，就好像本地进程会发出数据包一样。 因此，连接集群节点的网络必须能够路由 PodCIDR。

2. 配置native routing时，Cilium 会在 Linux 内核中自动启用 IP 转发。

![](/images/cilium02.png)

网络要求

- 运行 native routing时，运行 Cilium 的主机的网络必须能够使用分配给 pod 或其他工作负载的地址转发 IP 流量
- 节点上的 Linux 内核必须知道如何转发运行 Cilium 的所有节点的 pod 或其他工作负载的数据包。 这可以通过两种方式实现：
	- 节点本身不知道如何路由所有 pod IP，但网络上存在一个知道如何到达所有其他 pod 的路由器。 在这种情况下，Linux 节点配置为包含指向此类路由器的默认路由。 该模型用于云提供商网络集成。 有关更多详细信息，请参阅 Google Cloud、AWS ENI 和 Azure IPAM。
	- 每个单独的节点都知道所有其他节点的所有 Pod IP，并且路由被插入到 Linux 内核路由表中来表示这一点。 如果所有节点共享一个 L2 网络，则可以通过启用选项 auto-direct-node-routes: true 来解决这个问题。 否则，必须运行额外的系统组件（例如 BGP daemon）来分发路由。 请参阅 [使用 kube-router 运行 BGP 指南](https://docs.cilium.io/en/v1.11/gettingstarted/kube-router/#kube-router)，了解如何使用 kube-router 项目实现此目的。

### 配置方案

以下2个参数必须添加

- tunnel: disabled 启用native routing 模式
- ipv4-native-routing-cidr: x.x.x.x/y 设置可以执行native routing的 CIDR。

## 安装

>cilium 严重依赖内核，不同版本内核特性也不一样，一定要了解清楚内核版本

### Cilium 在k8s的运行方式

Cilium 在kubernetes安装 有2种运行模式，一种是完全替换kube-proxy， 如果底层 Linux 内核版本低，可以替换kube-proxy的部分功能，与原来的 kube-proxy 共存。

- `global.kubeProxyReplacement=strict`: 该选项期望使用无kube-proxy的Kubernetes设置，其中Cilium有望完全替代所有kube-proxy功能。 Cilium代理启动并运行后，将负责处理类型为ClusterIP，NodePort，ExternalIP和LoadBalancer的Kubernetes服务。如果不满足基本内核版本要求（[请参阅不带kube-proxy注释的Kubernetes](https://docs.cilium.io/en/v1.11/gettingstarted/kubeproxy-free/#kubeproxy-free)），则Cilium代理将在启动时退出并显示一条错误消息。
- `global.kubeProxyReplacement=probe` : 此选项适用于混合设置，即kube-proxy在Kubernetes集群中运行，其中Cilium部分替换并优化了kube-proxy功能。一旦Cilium agent启动并运行，它就会在基础内核中探查所需BPF内核功能的可用性，如果不存在，则依靠kube-proxy补充其余的Kubernetes服务处理，从而禁用BPF中的部分功能。在这种情况下，Cilium代理将向其日志中发送一条信息消息。例如，如果内核不支持 [Host-Reachable Services](https://docs.cilium.io/en/v1.7/gettingstarted/host-services/#host-services)，则节点的主机名空间的ClusterIP转换是通过kube-proxy的iptables规则完成的。
- `global.kubeProxyReplacement=partial`: 与探针类似，此选项用于混合设置，即kube-proxy在Kubernetes集群中运行，其中Cilium部分替换并优化了kube-proxy功能。与探查基础内核中可用的BPF功能并在缺少内核支持时自动禁用负责BPF kube-proxy替换的组件的探针相反，该部分选项要求用户手动指定应替换BPF kube-proxy的组件。与严格模式类似，如果不满足基本内核要求，则Cilium代理将在启动时通过错误消息进行援助。对于细粒度的配置，可以将`global.hostServices.enabled`，`global.nodePort.enabled`和`global.externalIPs.enabled`设置为true。默认情况下，所有三个选项均设置为false。

### Cilium 替换 Kube-Proxy

集群安装的时候

```js
kubeadm init --skip-phases=addon/kube-proxy
```

集群已经有kube-proxy

```js
kubectl -n kube-system delete ds kube-proxy
# Delete the configmap as well to avoid kube-proxy being reinstalled during a kubeadm upgrade (works only for K8s 1.19 and newer)
kubectl -n kube-system delete cm kube-proxy
# Run on each node with root permissions:
iptables-save | grep -v KUBE | iptables-restore
```

### Cilium各安装模式

helm 生成 template

```js
helm template cilium cilium/cilium --version 1.11.1 \
    --namespace kube-system \
    --set kubeProxyReplacement=strict \
    --set k8sServiceHost=apiserver.xxx.com \
    --set k8sServicePort=6443 > cilium.yaml
kubectl apply -f cilium.yaml
```

#### Cilium Host Routing Disabled with Legacy Mode[VxLAN]

要求

- kernel>= 4.9.17

```js
helm install cilium cilium/cilium --version 1.11.1 \
    --namespace kube-system \
    --set tunnel=vxlan \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=strict \
    --set prometheus.enabled=true \
    --set operator.prometheus.enabled=true \
    --set k8sServiceHost=apiserver.xxx.com \
    --set k8sServicePort=6443
```

参数说明：

- ipMasqAgent.enabled=true 代表集群Pod与nonMasqueradeCIDRs的地址通信时，不会主动masquerade，意味着不需要做SNAT
- prometheus.enabled=true 启用cilium-agent metrics
- operator.prometheus.enabled=true 启用cilium-operator metrics
- hubble.metrics.enabled 启用Hubble metrics

#### Cilium Host Routing Enabled with BPF Mode [VxLAN]

> kernel符合要求会自动启用host routing ，最低kernel >= 4.9.17

要求

- Kernel >= 5.10
- Direct-routing or tunneling
- eBPF-based 替换 kube-proxy
- eBPF-based masquerading

```js
helm install cilium cilium/cilium --version 1.11.1 \
    --namespace kube-system \
    --set tunnel=vxlan \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=strict \
    --set prometheus.enabled=true \
    --set operator.prometheus.enabled=true \
    --set k8sServiceHost=apiserver.xxx.com \
    --set k8sServicePort=6443
```

#### Cilium Host Routing Enabled with BPF Mode[Native Routing]

要求

- Kernel >= 5.10
- Direct-routing or tunneling
- eBPF-based 替换 kube-proxy
- eBPF-based masquerading

```js
helm install cilium cilium/cilium --version 1.11.1 \
    --namespace kube-system \
    --set tunnel=disabled \
    --set autoDirectNodeRoutes=true \
    --set kubeProxyReplacement=strict \
    --set loadBalancer.mode=hybrid \
    --set prometheus.enabled=true \
    --set operator.prometheus.enabled=true \
    --set nativeRoutingCIDR=10.0.0.0/8 \
    --set ipam.mode=kubernetes \
    --set ipam.operator.clusterPoolIPv4PodCIDR=10.0.0.0/8 \
    --set ipam.operator.clusterPoolIPv4MaskSize=24 \
    --set bpf.masquerade=true \
    --set bpf.hostRouting=false \
    --set k8sServiceHost=apiserver.xxx.com \
    --set k8sServicePort=6443
```

参数说明：

- `autoDirectNodeRoutes=true` 如果所有节点共享一个 L2 网络，则可以通过启用来实现Pod间的路由，此模式即DSR。 否则，必须运行其他系统组件（例如 BGP 守护程序）来分发路由
- `nativeRoutingCIDR` 设置可以执行 `Native Routing` 的 CIDR

#### Direct Server Return (DSR)

```js
helm install cilium cilium/cilium \
    --namespace kube-system \
    --set tunnel=disabled \
    --set autoDirectNodeRoutes=true \
    --set kubeProxyReplacement=strict \
    --set loadBalancer.mode=dsr \
    --set nativeRoutingCIDR=10.0.0.0/8 \
    --set ipam.mode=kubernetes \
    --set ipam.operator.clusterPoolIPv4PodCIDR=10.0.0.0/8 \
    --set ipam.operator.clusterPoolIPv4MaskSize=24 \
    --set bpf.masquerade=true \
    --set bpf.hostRouting=false \
    --set k8sServiceHost=apiserver.xxx.com \
    --set k8sServicePort=6443
```

#### Hybrid DSR and SNAT Mode

dsr 走 tcp, snat 走 udp

```js
helm install cilium cilium/cilium \
    --namespace kube-system \
    --set tunnel=disabled \
    --set autoDirectNodeRoutes=true \
    --set kubeProxyReplacement=strict \
    --set loadBalancer.mode=hybrid \
    --set k8sServiceHost=apiserver.xxx.com \
    --set k8sServicePort=6443
```

#### Bypass iptables Connection Tracking

对于无法使用 eBPF Host-Routing 并且因此网络数据包仍需要遍历主机命名空间中的常规网络堆栈的情况，iptables 会增加大量成本。 这种遍历成本可以通过禁用所有 Pod 流量的连接跟踪要求来最小化，从而绕过 iptables connection tracker。

要求

- Kernel >= 4.19.57, >= 5.1.16, >= 5.2
- Direct-routing configuration
- eBPF-based kube-proxy replacement
- eBPF-based masquerading or no masquerading

```js
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set installNoConntrackIptablesRules=true \
  --set kubeProxyReplacement=strict
```

具体最新1.11.3参数列表[参照](https://github.com/cilium/cilium/tree/v1.11.3/install/kubernetes/cilium)

官方文档

- [kube-router to run BGP](https://docs.cilium.io/en/latest/gettingstarted/kube-router/#kube-router)
- [Kubernetes Without kube-proxy​](https://docs.cilium.io/en/latest/gettingstarted/kubeproxy-free/)

[实践](https://www.cnblogs.com/apink/p/15200123.html)



