title: kubectl krew 离线安装插件
date: 2023-08-31 10:07:04
tags: [Kubernetes]
---

`krew` 离线安装：

官方安装文档：[https://krew.sigs.k8s.io/docs/user-guide/setup/install/](https://krew.sigs.k8s.io/docs/user-guide/setup/install/)

<!-- more -->

```sh
1. yum install git -y

2.  wget https://github.com/kubernetes-sigs/krew/releases/download/v0.4.4/krew-linux_amd64.tar.gz && tar zxvf krew-linux_amd64.tar.gz

3. 获取krew.yaml文件
wget https://github.com/kubernetes-sigs/krew-index/blob/master/plugins/krew.yaml

4. 查看krew.yaml  获取krew安装包地址
cat krew.yaml|grep krew-linux_amd64
          <td id="LC53" class="blob-code blob-code-inner js-file-line">  - <span class="pl-ent">uri</span>: <span class="pl-s">https://github.com/kubernetes-sigs/krew/releases/download/v0.4.3/krew-linux_amd64.tar.gz</span></td>

5. 下载 krew安装包
wget https://github.com/kubernetes-sigs/krew/releases/download/v0.4.4/krew-linux_amd64.tar.gz

6. 安装：
./krew-linux_amd64 install --manifest=krew.yaml --archive=krew-linux_amd64.tar.gz

7. 加载环境变量 vi ~/.bashrc
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

source ~/.bashrc

8. 验证
[root@i-u3hl7a5j ~]# kubectl krew  -h
krew is the kubectl plugin manager.
You can invoke krew through kubectl: "kubectl krew [command]..."

Usage:
  kubectl krew [command]

Available Commands:
  completion  generate the autocompletion script for the specified shell
  help        Help about any command
  index       Manage custom plugin indexes
  info        Show information about an available plugin
  install     Install kubectl plugins
  list        List installed kubectl plugins
  search      Discover kubectl plugins
  uninstall   Uninstall plugins
  update      Update the local copy of the plugin index
  upgrade     Upgrade installed plugins to newer versions
  version     Show krew version and diagnostics
```

离线安装 kubectl 插件 ns：

```sh
1. 查看插件列表 
https://github.com/kubernetes-sigs/krew-index/tree/master/plugins

2.  下载需要插件 ns 的yaml文件
wget https://github.com/kubernetes-sigs/krew-index/blob/master/plugins/ns.yaml

3.  获取 ns 插件所需要软件包
[root@i-u3hl7a5j tmp.KovSCCk8Ma]# cat ns.yaml |grep uri
    uri: https://github.com/ahmetb/kubectx/archive/v0.9.4.tar.gz

4. 下载
wget   https://github.com/ahmetb/kubectx/archive/v0.9.4.tar.gz

5. 安装
kubectl krew install --manifest=ns.yaml --archive=v0.9.4.tar.gz

6. 验证
[root@i-u3hl7a5j tmp.KovSCCk8Ma]# kubectl ns
default
ingress-nginx
kube-node-lease
kube-public
kube-system
```