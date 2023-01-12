title: CentOS7 部署K8S集群
date: 2021-04-25 19:34:16
tags: [Kubernetes]
---

部署规划

```js
10.211.55.18     k8s-node1
10.211.55.19     k8s-node2
10.211.55.20     k8s-node3
```

备注：第1步~第8步，所有的节点都要操作，第9、10步Master节点操作，第11步Node节点操作。 如果第9、10、11步操作失败，可以通过 kubeadm reset 命令来清理环境重新安装。
<!-- more -->
### 关闭防火墙

```js
$ systemctl stop firewalld
```

### 关闭selinux

```js
$ setenforce 0
```

### 关闭swap

```js
$ swapoff -a    临时关闭
$ free             可以通过这个命令查看swap是否关闭了
$ vim /etc/fstab  永久关闭
```

### 同步时间

```bash
yum install ntpdate
ntpdate cn.pool.ntp.org
```

### 添加主机名与IP对应的关系

```js
$ vim /etc/hosts
 添加如下内容：
10.211.55.18     k8s-node1
10.211.55.19     k8s-node2
10.211.55.20     k8s-node3
```

### 将桥接的IPV4流量传递到iptables 的链

```js
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sysctl --system
```

### 安装Docker

```js
1）下载并安装
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O/etc/yum.repos.d/docker-ce.repo
$ yum install docker-ce-20.10.17

2）配置镜像加速
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["http://hub-mirror.c.163.com", "https://docker.mirrors.ustc.edu.cn"]
}
EOF

3）设置开机启动
$ systemctl enable docker
$ systemctl start docker

4）查看Docker版本
$ docker --version
```

### 添加阿里云YUM软件源

```js
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 安装kubeadm，kubelet和kubectl

在部署kubernetes时，要求master node和worker node上的版本保持一致，否则会出现版本不匹配导致奇怪的问题出现。本文将介绍如何在CentOS系统上，使用yum安装指定版本的Kubernetes。
我们需要安装指定版本的kubernetes。那么如何做呢？在进行yum安装时，可以使用下列的格式来进行安装：

```js
yum install -y kubelet-<version> kubectl-<version> kubeadm-<version>
$ yum install -y kubelet-1.20.1 kubectl-1.20.1 kubeadm-1.20.1 kubernetes-cni-1.20.1
```

*如果出现错误提示，只需要清除yum缓存即可，然后再执行安装命令*

```
$ yum clean all 
```
使用yum安装程序时，提示xxx.rpm公钥尚未安装
使用 --nogpgcheck 命令格式跳过公钥检查，如下

```
$ yum install -y kubelet-1.20.1 kubectl-1.20.1 kubeadm-1.20.1 kubernetes-cni-1.20.1 --nogpgcheck

# 开机启动
$ systemctl enable kubelet
```

### 部署Kubernetes Master

#### 初始化kubeadm

```js
kubeadm init \
--apiserver-advertise-address=10.211.55.18 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.20.1 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16
```

当出现如下结果，表示初始化顺利

```
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.211.55.18:6443 --token m5fkxk.kf136eeya8v8ye5u \
    --discovery-token-ca-cert-hash sha256:3cf64ae72021e28adba5a7a99d2897b4f6a8a338997258027473cab4bf3d4f39
```

建议至少2 cpu ,2G，非硬性要求，1cpu，1G也可以搭建起集群。但是：1个cpu的话初始化master的时候会报 [WARNING NumCPU]: the number of available CPUs 1 is less than the required 2部署插件或者pod时可能会报warning：FailedScheduling：Insufficient cpu, Insufficient memory如果出现这种提示，说明你的虚拟机分配的CPU为1核，需要重新设置虚拟机master节点内核数。

查看镜像

```
$ docker images
```

#### 使用kubectl工具

复制如下命令直接执行

```js
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

下面就可以直接使用kubectl命令了

```
$ kubectl get node
```

#### 安装Pod网络插件（CNI）

- 安装插件

```js
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

- 查看是否部署成功

```js
$ kubectl get pods -n kube-system
```

- 再次查看node，可以看到状态为ready

```js
$ kubectl get node
```

安装失败了，执行如下命令，清理环境重新安装：

```js
$ kubeadm reset

$ rm -rf /etc/kubernetes
$ rm -rf /var/lib/etcd/
```

### Node节点加入集群

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：
复制上面命令，在node节点上执行

```js
kubeadm join 10.211.55.18:6443 --token m5fkxk.kf136eeya8v8ye5u \
    --discovery-token-ca-cert-hash sha256:3cf64ae72021e28adba5a7a99d2897b4f6a8a338997258027473cab4bf3d4f39
```

如果一直卡在 “Running pre-flight checks” 上，则很可能是时间未同步，token失效导致

第一步，检查master、node时间是否同步？

```js
$ date
执行如下命令同步时间
$ ntpdate time.nist.gov
```

第二步：如果还有其它错误如 Port 10250 is in use，执行如下命令

```js
$ kubeadm reset

$ rm -rf /etc/kubernetes
$ rm -rf /var/lib/etcd/
```

然后再重新执行kubeadm join ... 操作

执行成功后：

```js
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

再通过master节点查看node，发现node节点已以成功加入集群，并且Status状态为Ready

```js
NAME         STATUS   ROLES                  AGE    VERSION
k8s-node1   Ready    control-plane,master   13m    v1.20.1
k8s-node2    Ready    <none>                 4m8s   v1.20.1
```

#### 如果token忘记了，则可以通过如下操作：

- 查看token，如果token失效，则重新生成一个

```js
$ kubeadm token list
$ kubeadm token create
```

- 获取ca证书sha256编码hash值

```js
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

- 节点加入集群

```js
kubeadm join 10.211.55.18:6443 --token m5fkxk.kf136eeya8v8ye5u \
    --discovery-token-ca-cert-hash sha256:3cf64ae72021e28adba5a7a99d2897b4f6a8a338997258027473cab4bf3d4f39 
```

### 测试kubernetes集群

```js
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --``type``=NodePort
$ kubectl get pod,svc
```

```js
$ kubectl get pod,svc -o wide
```

通过浏览器访问：http://10.211.55.18:30328 可以正常访问

#### Node节点的添加和删除，及集群角色设定

- 添加node节点

```js
kubeadm join 10.211.55.18:6443 --token m5fkxk.kf136eeya8v8ye5u \
    --discovery-token-ca-cert-hash sha256:3cf64ae72021e28adba5a7a99d2897b4f6a8a338997258027473cab4bf3d4f39
```

- 执行成功后会在node节点的/etc/kubernetes目录下出创建kubelet.conf和pki等文件信息，同一节点再次加入时需要清空该目录下的文件信息

```
[root@centos-7-node ~]# cd /etc/kubernetes
[root@centos-7-node kubernetes]# pwd
/etc/kubernetes
[root@centos-7-node kubernetes]# ll
总用量 4
-rw-------. 1 root root 1909 4月  25 20:22 kubelet.conf
drwxr-xr-x. 2 root root    6 12月 19 01:28 manifests
drwxr-xr-x. 2 root root   20 4月  25 20:22 pki
[root@centos-7-node kubernetes]# 
```

- 在master节点查看集群节点信息

```js
[root@k8s-3 ~]# kubectl  get node
[root@centos-7-master ~]# kubectl  get node
NAME         STATUS   ROLES                  AGE   VERSION
k8s-node1   Ready    control-plane,master   33m   v1.20.1
k8s-node2    NotReady    <none>                 23m   v1.20.1
# 查看信息时，看到几个节点的状态都是NotReady，是因为还没有安装flannel插件，

# 等待几分钟后的信息，查看node节点的docker镜像，自动下载flannel镜像并创建网卡信息 
[root@centos-7-master ~]# kubectl  get node
NAME         STATUS   ROLES                  AGE   VERSION
k8s-node1   Ready    control-plane,master   33m   v1.20.1
k8s-node2    Ready    <none>                 23m   v1.20.1
```

- 设置node的角色

```js
[root@centos-7-master ~]# kubectl label nodes k8s-node1(节点名字)  node-role.kubernetes.io/node=
node/k8s-node1 labeled
[root@centos-7-master ~]# kubectl  get node
NAME         STATUS   ROLES                  AGE   VERSION
k8s-node1   Ready    control-plane,master   36m   v1.20.1
k8s-node2    Ready    node                   27m   v1.20.1
```

- 先将节点设置为维护模式(k8s-4是节点名称)

```js
[root@centos-7-master ~]# kubectl drain k8s-node1 --delete-local-data --ignore-daemonsets --force
```

- 删除k8s-node1节点

```js
[root@centos-7-master ~]# kubectl delete node k8s-node1
node "k8s-node1" deleted
[root@centos-7-master ~]# kubectl  get node
NAME         STATUS   ROLES                  AGE   VERSION
k8s-node1   Ready    control-plane,master   36m   v1.20.1
```

#### 将Pod调度到Master节点

- 出于安全考虑，默认配置下Kubernetes不会将Pod调度到Master节点。如果希望将k8s-master也当作Node使用，可以执行如下命令：

```js
kubectl taint node k8s-node1 node-role.kubernetes.io/master-
```

- 其中k8s-master是主机节点hostname如果要恢复Master Only状态，执行如下命令：

```js
kubectl taint node k8s-node1 node-role.kubernetes.io/master="":NoSchedule
```

*当只有一个master和一个node节点，在master可调度的情况下，node节点失联或者挂掉，node节点上的pod在k8s默认的检查时间之后并不会重新调度到master的情况*

```go
If the cluster contains 1 master and 1 worker and the nodeName of your master node is ending with "master"(e.g XXXmaster).
The pod will not be evicted from the worker node.

It is because the controller-manager does not take the node with nodeName ends with "master" as the worker node.
kubernetes/pkg/controller/nodelifecycle/node_lifecycle_controller.go

Lines 1004 to 1016 in 2adc8d7

 func legacyIsMasterNode(nodeName string) bool { 
 	// We are trying to capture "master(-...)?$" regexp. 
 	// However, using regexp.MatchString() results even in more than 35% 
 	// of all space allocations in ControllerManager spent in this function. 
 	// That's why we are trying to be a bit smarter. 
 	if strings.HasSuffix(nodeName, "master") { 
 		return true 
 	} 
 	if len(nodeName) >= 10 { 
 		return strings.HasSuffix(nodeName[:len(nodeName)-3], "master-") 
 	} 
 	return false 
 } 

If all worker nodes are NotReady, the controller-manager will enter master disruption mode.
Disruption mode ：#42733 (comment)

Around August last year we introduced a protection against master machine network isolation, which prevents any evictions from happening if master can't see any healthy Node. It then assumes that it's not a problem with Nodes, but with itself and just don't do anything.

I stop the kubelet on worker node and the controller-manager logs shows:

I0126 15:19:20.901414       1 node_lifecycle_controller.go:1230] Controller detected that all Nodes are not-Ready. Entering master disruption mode.
If you want your pod can be evicted from worker node in 1 master 1 worker cluster, you can:

set LegacyNodeRoleBehavior feature gate on controller-manager to false, for example, edit the controller-manager manifest, which is located at /etc/kubernetes/manifests/kube-controller-manager.yaml on your master node
...
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --feature-gates=LegacyNodeRoleBehavior=false  <-- add this line
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
...

rename the nodeName of your master node, do not ends with "master"
```

[Pods are not moved when Node in NotReady state](https://github.com/kubernetes/kubernetes/issues/55713)

[Kubernetes recreate pod if node becomes offline timeout](https://stackoverflow.com/questions/53641252/kubernetes-recreate-pod-if-node-becomes-offline-timeout)


### 证书过期

#### 查看证书过期时间

在 Master 节点上，执行 `kubeadm certs check-expiration` 命令，查看证书过期时间。

#### 更新证书

使用 ```kubeadm certs renew all``` 命令来更新证书。

#### 更新 ~/.kube/config 文件

```bash
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
sudo chmod 644 $HOME/.kube/config
```

#### kubelet client certificate

执行如下命令生成新的 `kubelet.conf`	 配置文件

```bash
# $NODE 表示集群中的节点名称
kubeadm kubeconfig user --org system:nodes --client-name system:node:$NODE --config=/etc/kubernetes/kubeadm-config.yaml > kubelet.conf

cp -i kubelet.conf /etc/kubernetes/kubelet.conf
systemctl restart kubelet
```

##### reboot node






