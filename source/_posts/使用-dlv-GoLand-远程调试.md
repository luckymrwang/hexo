title: 使用 dlv + GoLand 远程调试
date: 2021-07-16 10:04:08
tags: [Go]
---

### 前言

在开发/排查过程中, 偶尔会遇到一些仅在正式环境才能复现的BUG，但由于本地不能访问正式环境,，只能用有限的方式对问题进行Debug，例如添加日志等原始方法，效率过低，因此推荐借助IDE和Delve进行远程调试, 完成对BUG的快速排查与修复。

<!-- more -->
### 准备

- 一台可以访问正式环境的服务器

- IDE，这里推荐使用Jetbrains家的IDE，以下用GoLand为例

### 步骤

1、安装Delve

在服务器上安装Delve

官网: [https://github.com/derekparker/delve](https://github.com/derekparker/delve), 安装方法请参照官方wiki

2、查看IDE说明
程序的编译和启动尽量按照IDE的提示说明进行，在GoLand中点击菜单栏Run->EditConfigurations->+->GoRemote，我们将看到如下界面，中间为编译及启动说明

![](/images/goremote.png)

### 编译程序

根据IDE说明，在服务器上编译程序，必须添加`-gcflags`参数，给编译器传递-N -l参数，禁止编译器优化和内联

```go
go build -gcflags "all=-N -l" github.com/app/demo
```

### 使用dlv启动程序

#### 方式1

根据IDE说明, 在服务器上使用dlv启动编译好的程序

```go
dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient exec ./demo
```

如果程序需要启动参数, 则在后面添加--

```go
dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient exec ./demo -- -c=/config
# 等同于
./demo -c=/config
```

#### 方式2

获取服务器端运行的应用的pid

```go
ps aux | grep xxx
```

启动服务端应用的监听，命令如下：

```go
dlv attach PID --headless --api-version=2 --log --listen=:2345
```

### 本地配置Debug

在刚刚的`EditConfigurations`中添加`GoRemote`并在`Host`和`Port`中配置服务器IP地址和端口号

完成配置后，就可以运行建立 Debug。

