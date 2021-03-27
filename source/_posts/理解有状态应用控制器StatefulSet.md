title: 理解有状态应用控制器StatefulSet
date: 2021-03-27 16:47:21
tags: [Kubernetes]
---

### StatefulSet的设计原理

首先我们先来了解下Kubernetes的一个概念：有状态服务与无状态服务。

- 无状态服务（Stateless Service）：该服务运行的实例不会在本地存储需要持久化的数据，并且多个实例对于同一个请求响应的结果是完全一致的。这种方式适用于服务间相互没有依赖关系，如Web应用，在Deployment控制器停止掉其中的一个Pod不会对其他Pod造成影响。

- 有状态服务（Stateful Service）：服务运行的实例需要在本地存储持久化数据，比如数据库或者多个实例之间有依赖拓扑关系，比如：主从关系、主备关系。如果停止掉依赖中的一个Pod，就会导致数据丢失或者集群崩溃。这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为“有状态应用”（Stateful Application）。

其中无状态服务在我们前面文章中使用的Deployment编排对象已经可以满足，因为无状态的应用不需要很多要求，只要保持服务正常运行就可以，Deployment删除掉任意中的Pod也不会影响服务的正常，但面对相对复杂的应用，比如有依赖关系或者需要存储数据，Deployment就无法满足条件了，Kubernetes项目也提供了另一个编排对象StatefulSet。

<!--more-->
StatefulSet将有状态应用抽象为两种情况：

1、拓扑状态。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个 Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。

2、存储状态。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。它包含Deployment控制器ReplicaSet的所有功能，增加可以处理Pod的启动顺序，为保留每个Pod的状态设置唯一标识，同时具有以下功能：

- 稳定的、唯一的网络标识符
- 稳定的、持久化的存储
- 有序的、优雅的部署和缩放

### 有状态服务的拓扑状态
在上面我们提到有状态应用大致抽象为拓扑状态和存储状态，拓扑状态是为应用多实例中有相互依赖的服务而实现的。首先我们先来了解StatefulSet如何保证唯一的网络标识，这就需要涉及到Kubernetes 项目中非常实用的概念：Headless Service。

Service资源对象是k8s项目中用来将一组 Pod 暴露给外界访问的一种机制。

Service也分为两种方式分别是：

- 第一种方式，是以 Service 的 VIP（Virtual IP，即：虚拟 IP）方式。比如：当我访问 10.0.23.1 这个 Service 的 IP 地址时，10.0.23.1 其实就是一个 VIP，它会把请求转发到该 Service 所代理的某一个 Pod 上，相当于前面的负载均衡代理。
- 第二种方式，就是以 Service 的 DNS 方式。比如：这时候，只要我访问“my-svc.my-namespace.svc.cluster.local”这条 DNS 记录，就可以访问到名叫 my-svc 的 Service 所代理的某一个 Pod。

而第二种以DNS的方式又分为两种处理方法：

- Normal Service：正常使用，访问“my-svc.my-namespace.svc.cluster.local”解析到的，正是 my-svc 这个 Service 的 VIP，然后将请求通过VIP地址转发到代理的Pod。
- Headless Service：无头服务，访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是 my-svc 代理的某一个 Pod 的 IP 地址。可以看到，这里的区别在于，Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。

定义Headless Service 资源文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

Headless Service 在配置中和普通的Service的yaml文件定义基本相同，只是修改了clusterIP 字段为None，这样Headless Service 就不需要分配VIP地址，可以直接通过DNS的方式直接访问到Pod的IP地址。创建Headless Service 后它所代理的所有 Pod 的 IP 地址，都会被绑定一个这样格式的 DNS 记录：

```yaml
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

这个 DNS 记录，正是 Kubernetes 项目为 Pod 分配的唯一的“可解析身份”（Resolvable Identity）。有了这个“可解析身份”，只要你知道了一个 Pod 的名字，以及它对应的 Service 的名字，就可以通过这条 DNS 记录访问到 Pod 的 IP 地址。

定义StatefulSet资源文件

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
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
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```

上面这个YAML文件比我们之前部署的nginx-deployment增加了ServiceName:nginx的字段定义，即：告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候，请使用 nginx 这个 Headless Service 来保证 Pod 的“可解析身份”。

下面我们创建这些资源：

```yaml
$ kubectl create -f svc.yaml
$ kubectl get service nginx
NAME      TYPE         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     ClusterIP    None         <none>        80/TCP    10s
$ kubectl create -f statefulset.yaml
$ kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web       2         1         19s
$ kubectl get pods -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          12m
web-1   1/1     Running   0          10m
```

可以看到，Pod的名字后面都以数字顺序为后缀，这个是因为StatefulSet 对它所管理 Pod 名字进行了编号，编号规则是：-，编号是从 0 开始累加，与 StatefulSet 的每个 Pod 实例一一对应，不会重复。这些 Pod 的创建，也是严格按照编号顺序进行的。比如，在 web-0 进入到 Running 状态、并且细分状态（Conditions）成为 Ready 之前，web-1 会一直处于 Pending 状态。

当这两个 Pod 都进入了 Running 状态之后，就可以查看到它们各自唯一的“网络身份”了。我们使用kubectl exec命令分别进入到容器中查看它们的 hostname。

```yaml
$ kubectl exec web-0 -- sh -c 'hostname' 
web-0
$ kubectl exec web-1 -- sh -c 'hostname' 
web-1
```

然后我们启动一个一次性的pod来验证是否可以通过DNS解析到Pod的IP地址

```yaml
$ kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.7

$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.7
```

可以看到我们解析web-0.nginx这个DNS已经正确的返回了Pod的IP地址，现在我们再打开一个终端来删除这两个Pod，看看会发生什么样的变化？

```yaml
$ kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```

查看Statefulset重新创建的Pod是否可以正常解析？

```yaml
$ kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh 
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.8

$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.8
```

可以发现重新创建的pod也是按照原先的顺序来创建的，虽然Pod的IP地址有变化，但是用之前的DNS解析还是没有变化的，保持了原先的唯一网络标识，StatefulSet 也就保证了 Pod 网络标识的稳定性。至此Kubernetes 就成功地将 Pod 的拓扑状态（比如：哪个节点先启动，哪个节点后启动），按照 Pod 的“名字 + 编号”的方式固定了下来。

### 有状态服务的存储状态

下面我们来继续探究StatefulSet对存储状态的管理机制，在前面我们创建Pod需要使用存储的时候，只需要在资源文件中添加spec.volumes字段声明使用volume就可以，比如设置为hostpath或者emptyDir 。但实际环境中开发人员并不清楚我们那些Volume可以使用，所以存储我们就需要使用Kubernetes的另一个资源对象PVC（Persistent Volume Claim）的功能。

- PVC 的全称是：PersistentVolumeClaim（持久化卷声明），PVC 是用户存储的一种声明，PVC 和 Pod 比较类似，Pod 消耗的是节点，PVC 消耗的是 PV 资源，Pod 可以请求 CPU 和内存，而 PVC 可以请求特定的存储空间和访问模式。对于真正使用存储的用户不需要关心底层的存储实现细节，只需要直接声明使用 PVC 即可，不会暴露后端存储的信息。
- PV 的全称是：PersistentVolume（持久化卷），是对底层的共享存储的一种抽象，PV 由运维人员进行创建和配置，它和具体的底层的共享存储技术的实现方式有关，比如 Ceph、GlusterFS、NFS 等，都是通过插件机制完成与共享存储的对接。
PVC消耗的是PV资源，PV消耗的是后端共享存储。当我们创建一个PVC时，kubernetes 就会自动为PVC绑定一个符合条件的 PV，这里我们PV我们后续会在持久化存储文章中详细讲解，在这个例子中不做过多描述，这里我们已经提前创建完成。

继续以上面拓扑状态的应用为例，我们在资源文件中声明使用PVC：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
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
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce  # Volume 的挂载方式是可读写，并且只能被挂载在一个节点上而非被多个节点共享。
      resources:
        requests:
          storage: 1Gi  #Volume 大小至少是 1 GiB。
```

我们为这个 StatefulSet 额外添加了一个volumeClaimTemplates字段。它跟 Deployment 里 Pod 模板（PodTemplate）的作用类似。也就是说，凡是被这个 StatefulSet 管理的 Pod，都会声明一个对应的 PVC；而这个 PVC 的定义，就来自于 volumeClaimTemplates这个模板字段。更重要的是，这个 PVC 的名字，会被分配一个与这个 Pod 完全一致的编号。其中这个自动创建的 PVC，与 PV 绑定成功后，就会进入 Bound 状态，这就意味着这个 Pod 可以挂载并使用这个 PV 了。

我们来创建这个资源对象：

```yaml
$ kubectl create -f statefulset.yaml
$ kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-15c268c7-b507-11e6-932f-42010a800002   1Gi        RWO           48s
www-web-1   Bound     pvc-15c79307-b507-11e6-932f-42010a800002   1Gi        RWO           48s
```

可以看到PVC的命名方式为<PVC名字>-<StatefulSet名字>-<Pod编号>，PVC的状态已经显示为Bound，说明已经绑定到符合条件的PV，现在我们测试验证下数据是否会丢失 ？

分别往Pod的Volume目录中写入内容为hostname的index.html文件，然后分别访问pod的Nginx进程，查看返回信息是否正确

```yaml
$ for i in 0 1; do kubectl exec web-$i -- sh -c 'echo hello $(hostname) > /usr/share/nginx/html/index.html'; done
$ for i in 0 1; do kubectl exec -it web-$i -- curl localhost; done
hello web-0
hello web-1
```

然后我们手动删除这两个Pod

```yaml
$ kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```

被删除之后，这两个 Pod 会被按照编号的顺序被重新创建出来。然后我们在新创建的容器里通过访问“http://localhost”的方式去访问 web-0 里的 Nginx 服务：

```yaml
# 在被重新创建出来的Pod容器里访问http://localhost
$ kubectl exec -it web-0 -- curl localhost
hello web-0
```

可以发现请求依然会返回：“hello web-0”，也就是说，原先与：“web-0” 的 Pod 绑定的 PV，在这个 Pod 被重新创建之后，依然和新的“ web-0 ” Pod 绑定在了一起。对于 Pod web-1 来说，也是完全一样的情况。

这是因为我们删除pod后，之前绑定的PV和PVC并不会删除，数据仍然保存在后端的远程存储如Ceph中，StatefulSet发现Pod消失后，会自动创建一个新的Pod，名字编号也是和之前相同，而这个新的Pod声明使用的还是之前的PVC名字，所以在这个新的 Pod 创建出来之后，Kubernetes 会为它查找之前绑定的PVC ，就会直接找到旧 Pod 遗留下来的同名的 PVC，进而找到跟这个 PVC 绑定在一起的 PV。这样新的 Pod 就可以挂载到旧 Pod 对应的那个 Volume，并且获取到保存在 Volume 里的数据。通过这种方式，Kubernetes 的 StatefulSet 就实现了对应用存储状态的管理。

### 使用StatefulSet控制器部署ES集群

#### 创建无头服务

编写elasticsearch-svc.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
```

创建服务资源对象

```yaml
$ kubectl create -f elasticsearch-svc.yaml
$ kubectl  get svc -n logging
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
elasticsearch   ClusterIP   None            <none>        9200/TCP,9300/TCP   13d
```

#### 部署StorageClass持久化存储

集群使用NFS 作为后端存储资源，在主节点安装NFS，共享/data/k8s/目录。

```yaml
$ systemctl stop firewalld.service
$ yum -y install nfs-utils rpcbind
$ mkdir -p /data/k8s
$ chmod 755 /data/k8s
$ vim /etc/exports
/data/k8s  *(rw,sync,no_root_squash)
$ systemctl start rpcbind.service
$ systemctl start nfs.service
```

创建nfs-client 的自动配置程序Provisioner， nfs-client.yaml

```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 172.16.1.100
            - name: NFS_PATH
              value: /data/k8s
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.16.1.100
            path: /data/k8s
```

创建ServiceAccount，然后绑定集群对应的操作权限：nfs-client-sa.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

创建StorageClass，elasticsearch-storageclass.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: es-data-db
provisioner: fuseim.pri/ifs
```

部署服务资源对象

```yaml
$ kubectl create -f nfs-client.yaml
$ kubectl create -f nfs-client-sa.yaml
$ kubectl create -f elasticsearch-storageclass.yaml 
$ kubectl  get pods 
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-5b486d9c65-9fzjz   1/1     Running   9          13d
$ kubectl get storageclass
NAME                PROVISIONER      AGE
es-data-db          fuseim.pri/ifs   13d
```

### 使用StatefulSet 部署Es Pod

Elasticsearch 需要稳定的存储来保证 Pod 在重新调度或者重启后的数据依然不变，所以我们需要使用 StatefulSet 控制器来管理 Pod。

编写elasticsearch-statefulset.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet    
metadata:
  name: es   #定义了名为 es 的 StatefulSet 对象
  namespace: logging
spec:
  serviceName: elasticsearch  #和前面创建的 Service 相关联，这可以确保使用以下 DNS 地址访问 StatefulSet 中的每一个 Pod：es-[0,1,2].elasticsearch.logging.svc.cluster.local，其中[0,1,2]对应于已分配的 Pod 序号。
  replicas: 3  #3个副本
  selector:    #设置匹配标签为app=elasticsearch
    matchLabels:
      app: elasticsearch    
  template:    #定义Pod模板
    metadata:
      labels: 
        app: elasticsearch
    spec: 
      initContainers:  #初始化容器，在主容器执行前运行
      - name: increase-vm-max-map  #第一个Init容器用来增加操作系统对mmap计数的限制
        image: busybox  
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit   #第二个Init容器用来执行ulimit命令，增加打开文件描述符的最大数量
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
      containers:   
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
        ports:
        - name: rest
          containerPort: 9200
        - name: inter
          containerPort: 9300
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 1000m
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        env: #声明变量
        - name: cluster.name  # #Elasticsearch 集群的名称
          value: k8s-logs  
        - name: node.name #节点的名称，
          valueFrom: 
            fieldRef:
              fieldPath: metadata.name
        - name: cluster.initial_master_nodes
          value: "es-0,es-1,es-2"
        - name: discovery.zen.minimum_master_nodes   #将其设置为(N/2) + 1，N是我们的群集中符合主节点的节点的数量。我们有3个 Elasticsearch 节点，因此我们将此值设置为2（向下舍入到最接近的整数）。
          value: "2"
        - name: discovery.seed_hosts #设置在 Elasticsearch 集群中节点相互连接的发现方法。
          value: "elasticsearch"
        - name: ES_JAVA_OPTS  #设置为-Xms512m -Xmx512m，告诉JVM使用512 MB的最小和最大堆。您应该根据群集的资源可用性和需求调整这些参数。
          value: "-Xms512m -Xmx512m"
        - name: network.host
          value: "0.0.0.0"
  volumeClaimTemplates:   #持久化模板
  - metadata:
      name: data 
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ] #只能被 mount 到单个节点上进行读写
      storageClassName:  es-data-db
      resources:
        requests:
          storage: 100Gi
```

创建服务资源对象

```yaml
$ kubectl create -f elasticsearch-statefulset.yaml
statefulset.apps/es-cluster created
$ kubectl get pods -n logging
NAME                      READY     STATUS    RESTARTS   AGE
es-cluster-0              1/1       Running   0          20h
es-cluster-1              1/1       Running   0          20h
es-cluster-2              1/1       Running   0          20h
```

验证ES服务是否正常

将本地端口9200转发到 Elasticsearch 节点（如es-cluster-0）对应的端口：

```yaml
$ kubectl port-forward es-cluster-0 9200:9200 --namespace=logging
Forwarding from 127.0.0.1:9200 -> 9200
Forwarding from [::1]:9200 -> 9200
```

打开另一个终端请求es，出现以下内容则为部署成功。

```yaml
$ curl http://localhost:9200/_cluster/state?pretty

{
  "cluster_name" : "k8s-logs",
  "cluster_uuid" : "z9Hz1q9OS0G6GQBlWkOjuA",
  "version" : 19,
  "state_uuid" : "fjeJfNjjRkmFnX_1x_kzpg",
  "master_node" : "zcRrv4jnTfKFWGGdORpZKg",
  "blocks" : { },
  "nodes" : {
    "cqZH5iFOTYCKkNHiZK6uoQ" : {
      "name" : "es-0",
      "ephemeral_id" : "l3VAgaBYSLeY0wA9_CdWkw",
      "transport_address" : "192.168.85.195:9300",
      "attributes" : {
        "ml.machine_memory" : "8202764288",
        "ml.max_open_jobs" : "20",
        "xpack.installed" : "true"
      }
    },
    "zcRrv4jnTfKFWGGdORpZKg" : {
      "name" : "es-1",
      "ephemeral_id" : "LrD2UIReRfuIlGEArmwYuw",
      "transport_address" : "192.168.148.77:9300",
      "attributes" : {
        "ml.machine_memory" : "14543122432
-------
```

总结：

- StatefulSet 的控制器直接管理的是 Pod。通过在 Pod 的名字里加上事先约定好的编号，保证应用拓扑状态的服务稳定。
- Kubernetes 通过 Headless Service，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录，生成唯一的网络标识。
- StatefulSet 为每一个 Pod 分配并创建一个同样编号的 PVC。保证了每一个 Pod 都拥有一个独立的 Volume，保证数据不会丢失。

本文引自[这里](https://blog.csdn.net/qq_39254455/article/details/106278603)

