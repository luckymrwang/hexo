title: Golang安装及配置
date: 2016-05-25 14:04:43
tags: [Go]
toc: true
---

### Linux, Mac OS X, and FreeBSD tarballs

下载安装包解压到`/usr/local`, 例如：

	tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
<!-- more -->
*选择对应版本的安装包 go1.2.1.linux-amd64.tar.gz*

将`GOROOT`,`GOPATH`添加到环境变量

在linux下添加到 `$HOME/.bashrc` 中

*`.bashrc`与`.bash_profile`的区别：当shell是交互式登录shell时，读取.bash_profile文件，如在系统启动、远程登录或使用su -切换用户时；当shell是交互式登录和非登录shell时都会读取.bashrc文件，如：在图形界面中打开新终端或使用su切换用户时，均属于非登录shell的情况。
.bash_profile只在会话开始时被读取一次，而.bashrc则每次打开新的终端时，都会被读取。*

```
export GOROOT=/usr/local/go
export GOPATH=/data/plattech/go  // 项目目录 用go get *** 后会自动在该目录下创建src和pkg文件夹
export PATH=$PATH:$GOROOT/bin  // 如果安装了bee 则PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```