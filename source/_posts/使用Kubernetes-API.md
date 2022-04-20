title: 使用Kubernetes API
date: 2022-04-06 19:15:01
tags: [Kubernetes]
---

本文将基于cURL命令简单演示如何以REST的方式使用Kubernetes API，方便您使用开发语言原生的HTTPS方式操作Kubernetes集群。演示包括创建和删除Pod，创建和修改Deployment。

## 获取集群访问凭证KubeConfig

1. 登录容器服务管理控制台。
2. 在控制台左侧导航栏中，单击集群。
3. 在集群列表页面中，单击目标集群名称或者目标集群右侧操作列下的详情。
4. 单击连接信息页签，查看集群访问凭证（KubeConfig），将KubeConfig文件保存到本地。
5. 从KubeConfig文件中提取ca、key和apiserver信息，命令如下。

```sh
cat  ./kubeconfig |grep client-certificate-data | awk -F ' ' '{print $2}' |base64 -d > ./client-cert.pem
cat  ./kubeconfig |grep client-key-data | awk -F ' ' '{print $2}' |base64 -d > ./client-key.pem
APISERVER=`cat  ./kubeconfig |grep server | awk -F ' ' '{print $2}'`
```

<!-- more -->
## 使用cURL命令操作Kubernetes API

执行以下命令查看当前集群中所有Namespaces。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/api/v1/namespaces
```

- 常用的Pod相关操作。

执行以下命令查看default命名空间下的所有Pods。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/api/v1/namespaces/default/pods
```

执行以下命令创建Pod（JSON格式）。

```sh
cat nginx-pod.json
{
    "apiVersion":"v1",
    "kind":"Pod",
    "metadata":{
        "name":"nginx",
        "namespace": "default"
    },
    "spec":{
        "containers":[
            {
                "name":"ngnix",
                "image":"nginx:alpine",
                "ports":[
                    {
                        "containerPort": 80
                    }
                ]
            }
        ]
    }
}

--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/api/v1/namespaces/default/pods -X POST --header 'content-type: application/json' -d@nginx-pod.json
```

执行以下命令创建Pod（YAML格式）。

```sh
cat nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80

--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/api/v1/namespaces/default/pods -X POST --header 'content-type: application/yaml' --data-binary @nginx-pod.yaml
```

执行以下命令查询Pod状态。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/api/v1/namespaces/default/pods/nginx
```

执行以下命令查询Pod logs。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/api/v1/namespaces/default/pods/nginx/log
```

执行以下命令查询Pod的metrics数据（通过metric-server api）。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/apis/metrics.k8s.io/v1beta1/namespaces/default/pods/nginx
```

执行以下命令删除Pod。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/api/v1/namespaces/default/pods/nginx -X DELETE
```

- 常用的Deployment相关操作。

创建Deployment示例YAML文件如下。

```sh
cat nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image:  nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"

--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/apis/apps/v1/namespaces/default/deployments -X POST --header 'content-type: application/yaml' --data-binary @nginx-deploy.yaml
```

执行以下命令查看Deployment。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/apis/apps/v1/namespaces/default/deployments
```

执行以下命令更新Deployment（修改replicas副本数量）。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/apis/apps/v1/namespaces/default/deployments/nginx-deploy -X PATCH -H 'Content-Type: application/strategic-merge-patch+json' -d '{"spec": {"replicas": 4}}'
```

执行以下命令更新Deployment（修改容器镜像）。

```sh
--cert ./client-cert.pem --key ./client-key.pem -k $APISERVER/apis/apps/v1/namespaces/default/deployments/nginx-deploy -X PATCH -H 'Content-Type: application/strategic-merge-patch+json' -d '{"spec": {"template": {"spec": {"containers": [{"name": "nginx","image": "nginx:1.7.9"}]}}}}'
```

本文引自[这里](https://help.aliyun.com/document_detail/160530.html)