title: 搭建 eBPF 开发环境
date: 2022-08-19 21:45:28
tags: [eBPF]
---
### 前言
虽然 Linux 内核很早就已经支持了 eBPF，但很多新特性都是在 4.x 版本中逐步增加的。所以，想要稳定运行 eBPF 程序，内核至少需要 4.9 或者更新的版本。而在开发和学习 eBPF 时，为了体验和掌握最新的 eBPF 特性，我推荐使用更新的 5.x 内核。

作为 eBPF 最重大的改进之一，一次编译到处执行（简称 CO-RE）解决了内核数据结构在不同版本差异导致的兼容性问题。不过，在使用 CO-RE 之前，内核需要开启 `CONFIG_DEBUG_INFO_BTF=y` 和 `CONFIG_DEBUG_INFO=y` 这两个编译选项。为了避免你在首次学习 eBPF 时就去重新编译内核，我推荐使用已经默认开启这些编译选项的发行版，作为你的开发环境，比如：

- Ubuntu 20.10+
- Fedora 31+
- RHEL 8.2+
- Debian 11+

<!-- more -->

### 配置 eBPF 开发环境
可以到公有云平台上创建这些发行版的虚拟机，也可以借助 `Vagrant`、`Multipass` 等工具，创建本地的虚拟机。比如，使用我最喜欢的 `Vagrant` ，通过下面几步就可以创建出一个 `Ubuntu 21.10` 的虚拟机：

```bash
# 创建和启动Ubuntu 21.10虚拟机
vagrant init ubuntu/impish64
vagrant up
# 登录到虚拟机
vagrant ssh
```

虚拟机创建好之后，接下来就需要安装 eBPF 开发和运行所需要的开发工具，这包括：

1. 将 eBPF 程序编译成字节码的 LLVM；
2. C 语言程序编译工具 make；
3. 最流行的 eBPF 工具集 BCC 和它依赖的内核头文件；
4. 与内核代码仓库实时同步的 libbpf；
5. 同样是内核代码提供的 eBPF 程序管理工具 bpftool。

### CentOS Stream

根据官方文档，CentOS 8 已在 2021 年底被放弃，所以不再推荐将其作为生产环境继续使用。对于已有的用户来说，可以升级到 CentOS Stream 或 Rocky Linux 继续获得开源社区的支持。比如，你可以执行下面的命令，把 CentOS 8 升级为 CentOS Stream 8：

```bash
sudo dnf --disablerepo '*' --enablerepo extras swap centos-linux-repos centos-stream-repos -y
sudo dnf distro-sync -y
```

可以执行下面的命令，来安装 `eBPF` 这些必要的开发工具：

```bash
sudo yum install libbpf-devel make clang llvm elfutils-libelf-devel bpftool bcc-devel
```

如果已经熟悉了 Linux 内核的自定义编译和安装方法，并选择在旧的发行版中通过自行编译和升级内核搭建开发环境，上述的开发工具流程也需要做适当的调整。这里特别提醒下，libbpf-dev 这个库很可能需要从源码安装，具体的步骤可以参考 libbpf 的 [GitHub 仓库](https://github.com/libbpf/libbpf)。

####  安装 BCC

打开一个终端，SSH 连接到 CentOS Stream 8 系统后，执行 dnf info bcc-tools 查询 BCC 的版本，会看到如下的输出：

```bash
Available Packages
Name         : bcc-tools
Version      : 0.19.0
Release      : 5.el8
Architecture : x86_64
Size         : 448 k
Source       : bcc-0.19.0-5.el8.src.rpm
Repository   : appstream
Summary      : Command line tools for BPF Compiler Collection (BCC)
URL          : https://github.com/iovisor/bcc
License      : ASL 2.0
Description  : Command line tools for BPF Compiler Collection (BCC)
```

从输出中你可以发现，它自带的 BCC 版本是 `0.19.0`，而根据 BCC 的发布列表，其最新的版本已经到了 0.24.0。所以，为了使用较新的 BCC，从源码编译安装就是比直接使用 dnf 安装更好的选择。

在终端中执行下面的命令，我们就可以从源码编译和安装 `BCC 0.24.0` 版本：

```bash
# 第一步，安装必要的开发工具和开发库
sudo dnf makecache --refresh
sudo dnf groupinstall -y "Development tools"
sudo dnf install -y git bison flex cmake3 clang llvm bpftool elfutils-libelf-devel clang-devel llvm-devel ncurses-devel
# 第二步，从源码编译安装BCC
git clone https://github.com/iovisor/bcc.git
mkdir bcc/build; cd bcc/build
cmake -DENABLE_LLVM_SHARED=1 ..
make
sudo make install
cmake -DPYTHON_CMD=python3 .. # build python3 binding
pushd src/python/
make
sudo make install
popd
```

命令执行成功后，所有的 `BCC` 工具都会安装到 `/usr/share/bcc/tools` 目录下。比如，你可以执行 `sudo python3 /usr/share/bcc/tools/execsnoop` 命令来运行 `BCC` 自带的 `execsnoop` 工具。

####  安装 bpftrace

而对于另外一个常用的 `bpftrace` 来说，虽然也可以使用源码编译的方式安装，但实际上还有另外一个更简单的方式，那就是从 `bpftrace` 预先编译好的容器镜像中复制二进制文件。

我们执行下面的命令，安装容器工具 `podman` 之后，借助 `podman` 拉取 `bpftrace` 容器镜像，再将其中的 `bpftrace` 二进制文件复制出来，就可以把 `bpftrace` 安装到当前目录了：

```bash
# 第一步，安装podman
sudo dnf install -y podman
# 第二步，下载镜像后从中复制bpftrace二进制文件
podman pull quay.io/iovisor/bpftrace
podman run --security-opt label=disable -v $(pwd):/output quay.io/iovisor/bpftrace /bin/bash -c "cp /usr/bin/bpftrace /output"
```

这里需要你留意一点：在上面的命令中，我们使用了 `podman` 工具来拉取镜像并运行容器，这是因为 `CentOS Stream` 自带的软件包中不包含 `Docker`。

安装成功后，你可以执行下面的命令验证 `bpftrace` 的功能：

```bash
sudo ./bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
```

如果一切正常，你将会看到类似下面的输出：

```bash
Attaching 1 probe...
vmstats /proc/meminfo
vmstats /proc/stat
vminfo /var/run/utmp
...
```

到这里，我们就完成了 `CentOS Stream` 开发环境的配置。接下来要讲的 `Ubuntu 20.04` 的安装配置方法也是类似的，只是相应的软件包管理工具要换成 `apt` 系列工具。

### Ubuntu 20.04

首先，对于 BCC 的安装来说，由于 Ubuntu 系统中软件包的名字跟 CentOS 略有不同，所以在第一步安装开发工具和开发库时，我们需要做适当的调整。下面我们来看详细的安装步骤。

第一步，安装必要的开发工具和开发库：

```bash
sudo apt-get install -y  make clang llvm libelf-dev libbpf-dev libbpfcc-dev linux-tools-$(uname -r) linux-headers-$(uname -r)
```

如果已经熟悉了 Linux 内核的自定义编译和安装方法，并选择在旧的发行版中通过自行编译和升级内核搭建开发环境，上述的开发工具流程也需要做适当的调整。这里特别提醒下，libbpf-dev 这个库很可能需要从源码安装，具体的步骤可以参考 libbpf 的 [GitHub 仓库](https://github.com/libbpf/libbpf)。

####  安装 BCC

接下来的第二步是从源码编译安装 BCC，步骤跟上面的 CentOS Stream 是一样的，代码如下所示：

```bash
# 第二步，从源码编译安装BCC
git clone https://github.com/iovisor/bcc.git
mkdir bcc/build; cd bcc/build
cmake ..
make
sudo make install
cmake -DPYTHON_CMD=python3 .. # build python3 binding
pushd src/python/
make
sudo make install
popd
```

```bash
# For Lua support
sudo apt-get -y install luajit luajit-5.1-dev
```

同 CentOS Stream 系统一样，上述命令执行成功后，所有的 BCC 工具也会安装到 `/usr/share/bcc/tools` 目录下，你可以执行 `sudo python3 /usr/share/bcc/tools/execsnoop` 命令来验证 BCC 的安装。

####  安装 bpftrace

BCC 安装成功后，我们再来安装 `bpftrace`。由于 Ubuntu 已经自带了 Docker 软件包，因此你可以使用 `Docker`，通过 `bpftrace` 容器镜像来完成类似 `podman` 的安装步骤。具体的安装命令如下所示：

```bash
# 第一步，安装docker
sudo apt install -y docker.io
# 第二步，下载镜像后从中复制bpftrace二进制文件
sudo docker pull quay.io/iovisor/bpftrace
sudo docker run -v $(pwd):/output quay.io/iovisor/bpftrace /bin/bash -c "cp /usr/bin/bpftrace /output"
```

安装成功后，你可以执行同样的 `sudo ./bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'` 命令验证 `bpftrace` 的功能。

到这里，基本的开发环境就配置好了。不过环境的配置并没有完全结束，在使用 bpftool 时（比如执行命令 sudo bpftool prog dump jited id 2），你很可能会碰到 Error: No libbfd support 的错误。这说明发行版自带的 bpftool 默认不支持 libbfd，这时就需要我们下载内核源码重新编译安装。








