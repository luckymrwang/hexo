title: VSCode阅读Linux源码
date: 2022-09-09 13:52:15
tags: [Linux]
---
# 前言

平常都是通过网页在线查询和阅读 [Linux源码](https://elixir.bootlin.com/linux/latest/source) ，这个网站`bootlin`是一家提供在线查找阅读linux大部分内核源码的社区，支持查看不同的内核版本，特别方便。但是毕竟是网页版，每次跳转或者查看函数引用都要刷新页面，流畅度取决于使用者的网速，总归体验是不如在本地通过IDE查看的。所以才有了这篇操作手册式的文章，一是通过VSCode来流畅地查看linux源码，二是熟悉下搭建的流程。像Linux这种顶级而庞大的项目代码，本来以为会很麻烦，然而却非常地简单，毕竟VSCode足够强大。

<!-- more -->
先查看下最终效果图：

![](/images/linux_sock.png)

# 配置

## 基本环境

这里用宿主机 `Mac` + 虚拟机 `Ubuntu` 配置环境

### 下载并解压源码

源码从 [kernel.org](https://mirrors.edge.kernel.org/pub/linux/kernel/) 上下载，选择一个你想阅读的内核版本下载。
以 [linux-5.10.142.tar.gz](https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.10.142.tar.gz) 为列

在 Linux 虚拟机中执行如下命令下载内核代码：

```bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.10.142.tar.gz
```

使用如下命令解压到当前目录

```bash
tar -xzvf linux-5.10.142.tar.gz
```

## 配置 global 工具

### 安装 global 工具

global 工具是 GNU 协议下的源码标记软件。Ubuntu 上使用 apt 安装只需要执行命令：

```bash
sudo apt install global
```

### 安装 global 插件

VSCode 上有现成的插件可以直接使用，我们在 VSCode 这个 SSH 会话里安装 C/C++ GNU Global 插件，

![](/images/linux_gnu.png)

然后在内核代码项目中新建 `.vscode/settings.json`（如果自定义了 global 工具的路径的话需要在这里显式地配置 `gnuGlobal.globalExecutable` 和 `gnuGlobal.gtagsExecutable` 字段）。

![](/images/linux_ssh.png)

如果不清楚 global 的路径可以使用如下命令查看：

```bash
which is global
which is gtags
```

在 vscode 的 settings.json 配置里，指定 global 的相关路径。

```bash
"gnuGlobal.globalExecutable": "/usr/bin/global",
"gnuGlobal.gtagsExecutable": "/usr/bin/gtags",
"gnuGlobal.objDirPrefix": "/home/sino/go/src/linux/.global",
```

注意："gnuGlobal.objDirPrefix" 的路径必须要手动创建好，同时也要注意读写权限，如果不存在，会导致后续 Rebuild 的失败。

## 生成 tag

在 `VSCode` 工作区中按 `F1` 执行 `Global: Show GNU Global Version`，如果配置正确，右下角会显示 `global (GNU GLOBAL) <Global_Version>`。

再次执行 `Global: Rebuild Gtags Database`，等待完成后就可以愉快地阅读 Linux 源码了。





