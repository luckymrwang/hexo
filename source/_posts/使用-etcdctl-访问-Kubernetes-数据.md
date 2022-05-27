title: 使用 etcdctl 访问 Kubernetes 数据
date: 2022-05-06 16:01:16
tags: [Kubernetes]
---
### 前言

Kubenretes1.6 中使用 `etcd V3` 版本的 API，使用 `etcdctl` 直接 `ls` 的话只能看到 `/kube-centos` 一个路径。需要在命令前加上 `ETCDCTL_API=3` 这个环境变量才能看到 kuberentes 在 etcd 中保存的数据。

```js
ETCDCTL_API=3 etcdctl get /registry/namespaces/default -w=json|python -m json.tool
```

<!-- more -->
如果是使用 `kubeadm` 创建的集群，在 Kubenretes 1.11 中，`etcd` 默认使用 `tls` ，这时可以在 master 节点上使用以下命令来访问 `etcd` ：

```js
ETCDCTL_API=3 etcdctl --endpoints=<etcd-ip-1>:2379,<etcd-ip-2>:2379,<etcd-ip-3>:2379 \
--cacert=/etc/ssl/etcd/ssl/ca.crt \
--cert=/etc/ssl/etcd/ssl/apiserver-etcd-client.crt \
--key=/etc/ssl/etcd/ssl/apiserver-etcd-client.key \
get /registry/namespaces/default -w=json | jq .
```

- -w 指定输出格式

将得到这样的 json 的结果：

```json
{
    "count": 1,
    "header": {
        "cluster_id": 12091028579527406772,
        "member_id": 16557816780141026208,
        "raft_term": 36,
        "revision": 29253467
    },
    "kvs": [
        {
            "create_revision": 5,
            "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvZGVmYXVsdA==",
            "mod_revision": 5,
            "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlEmIKSAoHZGVmYXVsdBIAGgAiACokZTU2YzMzMDgtMWVhOC0xMWU3LThjZDctZjRlOWQ0OWY4ZWQwMgA4AEILCIn4sscFEKOg9xd6ABIMCgprdWJlcm5ldGVzGggKBkFjdGl2ZRoAIgA=",
            "version": 1
        }
    ]
}
```

使用 `--prefix` 可以看到所有的子目录，如查看集群中的 namespace：

```js
ETCDCTL_API=3 etcdctl get /registry/namespaces --prefix -w=json | python -m json.tool
```

### 对象数据

#### namespace

```js
/registry/namespaces/default
/registry/namespaces/game
/registry/namespaces/kube-node-lease
/registry/namespaces/kube-public
/registry/namespaces/kube-system
```

#### namespace级别对象

```js
/registry/{resource}/{namespace}/{resource_name}
```

以下以常见 k8s 对象为例：

```js
# deployment
/registry/deployments/default/game-2048
/registry/deployments/kube-system/prometheus-operator

# replicasets
/registry/replicasets/default/game-2048-c7d589ccf

# pod
/registry/pods/default/game-2048-c7d589ccf-8lsbw

# statefulsets
/registry/statefulsets/kube-system/prometheus-k8s

# daemonsets
/registry/daemonsets/kube-system/kube-proxy

# secrets
/registry/secrets/default/default-token-tbfmb

# serviceaccounts
/registry/serviceaccounts/default/default

# service
/registry/services/specs/default/game-2048

# endpoints
/registry/services/endpoints/default/game-2048
```

### 读取数据value

由于 k8s 默认 etcd 中的数据是通过 `protobuf` 格式存储，因此看到的 `key` 和 `value` 的值是一串字符串。

```js
# ETCDCTL_API=3 etcdctl get /registry/namespaces/test -w json | jq .
{
  "header": {
    "cluster_id": 12113422651334595000,
    "member_id": 8381627376898157000,
    "revision": 12321629,
    "raft_term": 20
  },
  "kvs": [
    {
      "key": "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvdGVzdA==",
      "create_revision": 11670741,
      "mod_revision": 11670741,
      "version": 1,
      "value": "azhzAAoPCgJ2MRIJTmFtZXNwYWNlElwKQgoEdGVzdBIAGgAiACokYWM1YmJjOTQtNTkxZi0xMWVhLWJiOTQtNmM5MmJmM2I3NmI1MgA4AEIICJuf3fIFEAB6ABIMCgprdWJlcm5ldGVzGggKBkFjdGl2ZRoAIgA="
    }
  ],
  "count": 1
}
```

其中 key 可以通过 `base64` 解码出来

```js
echo "L3JlZ2lzdHJ5L25hbWVzcGFjZXMvdGVzdA==" | base64 --decode

# output
/registry/namespaces/test
```

value 是值可以通过安装 [etcdhelper](https://github.com/openshift/origin/tree/master/tools/etcdhelper) 工具解析出来。

```js
# ./etcdhelper -key /etc/ssl/etcd/ssl/admin-centos-7-kubesphere.shared-key.pem -cert /etc/ssl/etcd/ssl/admin-centos-7-kubesphere.shared.pem -cacert /etc/ssl/etcd/ssl/ca.pem get /registry/namespaces/test
/v1, Kind=Namespace
{
  "kind": "Namespace",
  "apiVersion": "v1",
  "metadata": {
    "name": "test",
    "uid": "ac5bbc94-591f-11ea-bb94-6c92bf3b76b5",
    "creationTimestamp": "2020-02-27T05:11:55Z"
  },
  "spec": {
    "finalizers": [
      "kubernetes"
    ]
  },
  "status": {
    "phase": "Active"
  }
}
```

### 注意事项

- 由于 k8s 的 etcd 数据为了性能考虑，默认通过 `protobuf` 格式存储，不要通过手动的方式去修改或添加k8s数据。
- 不推荐使用json格式存储etcd数据，如果需要json格式，可以使用 `--storage-media-type=application/json` 参数存储，参考：[https://github.com/kubernetes/kubernetes/issues/44670](https://github.com/kubernetes/kubernetes/issues/44670)

### 快捷命令

由于`etcdctl`的命令需要添加很多认证参数和endpoints的参数，因此可以使用别名的方式来简化命令。

```js
# etcdctl 
alias ectl='ETCDCTL_API=3 etcdctl --endpoints=<etcd-ip-1>:2379,<etcd-ip-2>:2379,<etcd-ip-3>:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt  --key=/etc/kubernetes/pki/apiserver-etcd-client.key  --cert=/etc/kubernetes/pki/apiserver-etcd-client.crt'

# etcdhelper
alias ehelper='etcdhelper -key /etc/kubernetes/pki/apiserver-etcd-client.key -cert /etc/kubernetes/pki/apiserver-etcd-client.crt -cacert /etc/kubernetes/pki/etcd/ca.crt'
```

### etcdhelper的使用

etcdhelper 文档参考：[https://github.com/openshift/origin/tree/master/tools/etcdhelper](https://github.com/openshift/origin/tree/master/tools/etcdhelper)

```js
# 必要的认证参数
-key - points to master.etcd-client.key
-cert - points to master.etcd-client.crt
-cacert - points to ca.crt

# 命令操作参数
ls - list all keys starting with prefix
get - get the specific value of a key
dump - dump the entire contents of the etcd
```

示例

```js
$ ehelper ls /registry/leases/
/registry/leases/kube-node-lease/<ip-1>
/registry/leases/kube-node-lease/<ip-2>
/registry/leases/kube-node-lease/<ip-3>

$ ehelper get <key>
```

