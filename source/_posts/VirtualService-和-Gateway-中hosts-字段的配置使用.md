title: VirtualService 和 Gateway 中hosts 字段的配置使用
date: 2023-08-30 17:25:58
tags: [Istio]
---

Istio 中的 CR 资源 `VirtualService` 和 `Gateway` 都存在 `hosts` 属性，而且 `VirtualService.spec.http.route. destination.host` 也存在 `host` 字段，这些字段容易让人混淆，特以此做一解释和区分。

## VirtualService

### hosts

VirtualService 定义了一系列针对指定服务的流量路由规则。每个路由规则都针对特定协议的匹配规则。如果流量符合这些特征，就会根据规则发送到服务注册表中的目标服务（或者目标服务的子集或版本）。

<!-- more -->

VirtualService 配置示例：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx
  namespace: mesh
spec:
  hosts:
    - 172.16.85.174
    - nginx-test.mesh.svc.cluster.local
    - nginx-test
  gateways:
    - mesh-gw
  http:
    - route:
        - destination:
            host: nginx
            subset: v1
          weight: 20
        - destination:
            host: nginx2
            subset: v2
          weight: 80
```

使用 hosts 字段列举虚拟服务的主机——即用户指定的目标或是路由规则设定的目标。这是客户端向服务发送请求时使用的一个或多个地址。只有访问客户端的 Host 字段为 hosts 配置的地址才能路由到后端服务。

```js
hosts:

172.16.85.174
nginx-test.mesh.svc.cluster.local
nginx-test
```

虚拟服务主机名可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称（例如 Kubernetes 服务的短名称），隐式或显式地指向一个完全限定域名（FQDN）。也可以使用通配符（“*”）前缀，让您创建一组匹配所有服务的路由规则。

上面的 VirtualService 配置了多个 `hosts`，并且挂载了一个 `gateway`，客户端直接访问后端的 service 是可以通的，但是我们通过域名访问后端服务时候就需要指定 `host` 了。

首先我们直接访问下gateway域名，是无法访问通的，因为 `VirtualService` 流量规则指定了 `hosts`，我们的请求 `Host` 没在配置列表中。

```js
# curl nginx.istio.niewx.top
```

如果你希望能通过域名直接访问，可以将域名配置到 hosts 下，默认发起请求的 Host 就是域名本身

```yaml
spec:
  hosts:
    - 172.16.85.174
    - nginx-test.mesh.svc.cluster.local
    - nginx-test
    - nginx.istio.niewx.top
```

直接访问域名

```http
bash-4.4# curl -Iv nginx.istio.niewx.top
* Rebuilt URL to: nginx.istio.niewx.top/
*   Trying 10.0.0.183...
* TCP_NODELAY set
* Connected to nginx.istio.niewx.top (10.0.0.183) port 80 (#0)
> HEAD / HTTP/1.1
> Host: nginx.istio.niewx.top
> User-Agent: curl/7.61.1
> Accept: */*
> 
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< server: envoy
server: envoy
< date: Mon, 13 Sep 2021 10:34:12 GMT
date: Mon, 13 Sep 2021 10:34:12 GMT
< content-type: text/html
content-type: text/html
< content-length: 15
content-length: 15
< last-modified: Mon, 13 Sep 2021 05:50:32 GMT
last-modified: Mon, 13 Sep 2021 05:50:32 GMT
< etag: "613ee6a8-f"
etag: "613ee6a8-f"
< accept-ranges: bytes
accept-ranges: bytes
< x-envoy-upstream-service-time: 12
x-envoy-upstream-service-time: 12

< 
* Connection #0 to host nginx.istio.niewx.top left intact
```

### host

上例中的 `VirtualService` 指定的路由中有两个 subnet ，一个是 `host: nginx` ，另一个是 `host: nginx2`。这里的 host 值指对应的 kubernetes 中的 service 的名称，即流量目标对应的服务。

如果查找失败，则丢弃流量。Kubernetes 用户注意：当使用服务的短名称时（例如使用 `reviews`，而不是 `reviews.default.svc.cluster.local`），Istio 会根据规则所在的命名空间来处理这一名称，而非服务所在的命名空间。假设 `default` 命名空间的一条规则中包含了一个 `reivews` 的 `host` 引用，就会被视为 `reviews.default.svc.cluster.local`，而不会考虑 `reviews` 服务所在的命名空间。为了避免可能的错误配置，建议使用 FQDN 来进行服务引用。

## Gateway

Gateway 描述了一个负载均衡器，用于承载网格边缘的进入和发出连接。这一规范中描述了一系列开放端口，以及这些端口所使用的协议、负载均衡的 SNI 配置等内容。

下面是一个网关资源的例子：

```yaml
 apiVersion: networking.istio.io/v1alpha3
 kind: Gateway
 metadata:
   name: my-gateway
   namespace: default
 spec:
   selector:
     istio: ingressgateway
   servers:
   - port:
       number: 80
       name: http
       protocol: HTTP
     hosts:
     - dev.example.com
     - test.example.com
```

上述网关资源设置了一个代理，作为一个负载均衡器，为入口暴露 80 端口。网关配置被应用于 Istio 入口网关代理，我们将其部署到 `istio-system` 命名空间，并设置了标签 `istio: ingressgateway`。通过网关资源，我们只能配置负载均衡器。`hosts` 字段作为一个过滤器，只有以 `dev.example.com` 和 `test.example.com` 为目的地的流量会被允许通过。

为了控制和转发流量到集群内运行的实际 Kubernetes 服务，我们必须用特定的主机名（例如 `dev.example.com` 和 `test.example.com`）配置一个 `VirtualService`，然后将网关连接到它。

![](/images/vs_gw.png)

如上图，每个 CR `Gateway` 可以绑定多个 CR `VirtualService` ：

- CR `VirtualService` 通过 `gateways` 字段中的值与 CR `Gateway` 绑定
- CR `Gateway` 通过 `hosts` 字段中的匹配的 `host` 值选择对应的 CR `VirtualService`

参考：

- [创建部署Gateway并使用网关暴露服务](https://www.cnblogs.com/renshengdezheli/p/16838966.html)