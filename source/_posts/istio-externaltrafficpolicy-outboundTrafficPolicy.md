title: istio externaltrafficpolicy outboundTrafficPolicy
date: 2021-04-15 19:37:26
tags: [Istio]
---

### outboundTrafficPolicy

服务网格中的 Pod 的所有出站流量都会重定向到其 Sidecar 代理，集群外部 URL 的可访问性取决于代理的配置。如果 outboundTrafficPolicy 配置为 REGISTRY_ONLY， Istio 代理会阻止任何没有在网格中定义的 HTTP 服务或 service entry 主机的访问。

三个解决方案：
<!-- more -->
1、允许 Envoy 代理将请求传递到未在网格内配置过的服务。

```yaml
outboundTrafficPolicy:
    mode: ALLOW_ANY
```
2、配置 service entries 以提供对外部服务的受控访问。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: baidu
spec:
  hosts:
  - www.baidu.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```
3、对于特定范围的 IP，完全绕过 Envoy 代理。

```yaml
spec:
   excludeIPRanges:10.244.0.0/16
```

具体的官网文档：[Accessing External Services](https://link.zhihu.com/?target=https%3A//istio.io/latest/docs/tasks/traffic-management/egress/egress-control/)

### externaltrafficpolicy

背景：部署在istio的应用A调用浏览器内核直接通过域名访问应用B并截图，但域名访问链接不可达（或链接拒绝）

产生原因：externaltrafficpolicy 设置为local，1.源地址信息丢失；2.本节点pod的请求在本节点消耗，如果本节点内pod没有对应服务，则连接丢失。

![istio-traffic-policy](/images/istio-trafic-policy.png)

图片来源：[【华为云技术分享】K8s中的external-traffic-policy是什么？_华为云官方博客-CSDN博客](https://link.zhihu.com/?target=https%3A//blog.csdn.net/devcloud/article/details/105657334)

解决办法：externaltrafficpolicy: cluster,但pod内日志记录不了用户的原始地址。




