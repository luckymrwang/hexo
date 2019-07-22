title: Minikube的安装
date: 2019-07-22 11:56:21
tags: [Minikube]
---

### 安装Minikube可执行程序

#### MAC

```
brew cask install minikub
```

#### Linux

```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

<!--more-->
#### Windows

```
https://storage.googleapis.com/minikube/releases/latest/minikube-windows-amd64.exe 
```

*下载后，重命名成minikube.exe,然后把这个文件的所在目录添加到系统环境变量PATH里*

安装完以后，我们可以通过minikube version 查看系统版本

```
~ minikube version

minikube version: v1.2.0
```

### 安装kubectl

#### MAC

```
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/darwin/amd64/kubectl
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

#### Linux

```
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
$ chmod +x ./kubectl
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

#### Windows

```
https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/windows/amd64/kubectl.exe
```

*添加到系统PATH环境变量里*

```
~ kubectl version

Client Version: version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.11", GitCommit:"637c7e288581ee40ab4ca210618a89a555b6e7e9", GitTreeState:"clean", BuildDate:"2018-11-26T14:38:32Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0", GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean", BuildDate:"2019-06-19T16:32:14Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"linux/amd64"}
```

### Hyperkit driver

Install the hyperkit VM manager using brew:

```
brew install hyperkit
```

Then install the most recent version of minikube's fork of the hyperkit driver:

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-hyperkit \
&& sudo install -o root -g wheel -m 4755 docker-machine-driver-hyperkit /usr/local/bin/
```

To use the driver:

```
minikube start --vm-driver hyperkit
```

or, to use hyperkit as a default driver for minikube:

```
minikube config set vm-driver hyperkit
```

### 运行minikube程序创建k8s

通过 minikube starat 去创建k8s环境。

```
~ minikube start

* minikube v1.2.0 on darwin (amd64)
* using image repository registry.cn-hangzhou.aliyuncs.com/google_containers
* Tip: Use 'minikube start -p <name>' to create a new cluster, or 'minikube delete' to delete this one.
* Re-using the currently running hyperkit VM for "minikube" ...
* Waiting for SSH access ...
* Configuring environment for Kubernetes v1.15.0 on Docker 18.09.6
* Relaunching Kubernetes v1.15.0 using kubeadm ...
* Verifying: apiserver proxy etcd scheduler controller dns
* Done! kubectl is now configured to use "minikube"
```

通过kubectl cluster-info 去连一下k8s api server.

```
~ kubectl cluster-info

Kubernetes master is running at https://192.168.64.3:8443
KubeDNS is running at https://192.168.64.3:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

*此时并不代表，整个k8s集群搭建好了，因为k8s里的服务还需要起，比如API server，scheduler，kubelet等等，它们都是以容器的方式在后台启动。可以通过`minikube ssh`进到虚机里，然后看看是否有一些container运行起来了*

然后退出来，在本地运行：

```
minikube dashboard
```

会在本地弹出浏览器，就是Kubernetes的dashboard。







