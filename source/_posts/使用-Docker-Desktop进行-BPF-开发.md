title: 使用 Docker Desktop进行 BPF 开发
date: 2022-05-23 15:36:20
tags: [eBPF]
---

Docker Desktop 是 Windows 和 Mac 上最为流行 Docker 开发环境，是否有办法在Docker Desktop中，利用容器来使用eBPF呢？本文参考了部分 [https://github.com/singe/ebpf-docker-for-mac](https://github.com/singe/ebpf-docker-for-mac?spm=a2c6h.12873639.article-detail.4.3e805798KLBkfr) 相关实现，来帮助大家尝试一下。

<!-- more -->

![image](/images/macebpf.jpeg)

## bcc & bpftrace

Docker Desktop for Mac 通过一个虚拟机，来运行基于[Linuxkit](https://github.com/linuxkit/linuxkit?spm=a2c6h.12873639.article-detail.5.3e805798KLBkfr)构建的操作系统支持Docker环境。我们无法直接访问Virtual Machine，我们需要在 Docker容器中运行 eBPF 工具, 这需要有如下的前提条件：

- /usr/src/ 需要包含内核源代码
- debugfs 被正确挂载。 `mount -t debugfs debugfs /sys/kernel/debug`
- /lib/modules/ 需要挂载 host 宿主机上相关目录
- 需要在特权方式运行，比如 `docker run --privileged ...`
- 需要使用宿主机 PID 名空间，比如 `docker run --pid=host ...`

我们首先获取当前宿主机内核版本信息

```bash
$ docker run -it --rm --privileged --pid=host justincormack/nsenter1
# uname -r
5.10.47-linuxkit
```

在Docker Hub上，Docker 在 [docker/for-desktop-kernel](https://hub.docker.com/r/docker/for-desktop-kernel/tags?spm=a2c6h.12873639.article-detail.6.3e805798KLBkfr) 仓库中发布了 Docker Desktop 所包含的 linuxkit 内核代码的容器镜像。大家根据上面的内核版本信息就能定位相应的镜像 tag。

然后，我们来构建属于自己的 Docker 镜像，比如我希望构建一个Docker镜像包含，[bcc](https://github.com/iovisor/bcc?spm=a2c6h.12873639.article-detail.7.3e805798KLBkfr) 和 [bpftrace](https://github.com/iovisor/bpftrace?spm=a2c6h.12873639.article-detail.8.3e805798KLBkfr) 等eBPF开发工具。我们创建如下 Dockerfile.tools 来构建相应镜像

```docker
FROM docker/for-desktop-kernel:5.10.47-0b705d955f5e283f62583c4e227d64a7924c138f AS ksrc

FROM ubuntu:20.04 AS bpftrace
COPY --from=ksrc /kernel-dev.tar /
RUN tar xf kernel-dev.tar && rm kernel-dev.tar
# Use Alibaba Cloud mirror for ubuntu
RUN sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/' /etc/apt/sources.list
# Install LLVM 10.0.1
RUN apt-get update && apt install -y wget lsb-release software-properties-common && wget https://apt.llvm.org/llvm.sh && chmod +x llvm.sh && ./llvm.sh 10
ENV PATH "$PATH:/usr/lib/llvm-10/bin"

# Build/Install bpftrace
RUN apt-get install -y bpftrace

# Build/Install bcc
WORKDIR /root
RUN DEBIAN_FRONTEND="noninteractive" apt install -y kmod vim bison build-essential cmake flex git libedit-dev \
  libcap-dev zlib1g-dev libelf-dev libfl-dev python3.8 python3-pip python3.8-dev clang libclang-dev && \
  ln -s $(which python3) /usr/bin/python
RUN git clone https://github.com/iovisor/bcc.git && \
    mkdir bcc/build && \
    cd bcc/build && \
    cmake .. && \
    make && \
    make install && \
    cmake -DPYTHON_CMD=python3 .. && \
    cd src/python/ && \
    make && \
    make install && \
    sed -i "s/self._syscall_prefixes\[0\]/self._syscall_prefixes\[1\]/g" /usr/lib/python3/dist-packages/bcc/__init__.py

CMD mount -t debugfs debugfs /sys/kernel/debug && /bin/bash
```

运行如下命令，构建镜像

```docker
$ docker build -t ebpf-for-mac -f ./Dockerfile.tools .
[+] Building 6097.8s (16/16) FINISHED
...
```

或者可以直接拉取以构建好的镜像，

```docker
$ docker pull registry.cn-hangzhou.aliyuncs.com/denverdino/ebpf-for-mac
$ docker tag registry.cn-hangzhou.aliyuncs.com/denverdino/ebpf-for-mac ebpf-for-mac
```

运行如下命令，来通过Docker镜像运行 bcc 测试应用。

```docker
$ docker run -it --rm \
  --name ebpf-for-mac \
  --privileged \
  -v /lib/modules:/lib/modules:ro \
  -v /etc/localtime:/etc/localtime:ro \
  --pid=host \
  ebpf-for-mac
  
# wget https://raw.githubusercontent.com/singe/ebpf-docker-for-mac/main/hello_world.py

# python3 hello_world.py
b'      kube-proxy-5454    [003] d... 10679.347316: bpf_trace_printk: Hello world'
b''
b'      kube-proxy-5469    [002] d... 10679.355387: bpf_trace_printk: Hello world'
b''
b'         dockerd-1807    [001] d... 10680.178545: bpf_trace_printk: Hello world'
b''
b'            runc-95361   [003] d... 10680.191472: bpf_trace_printk: Hello world'
b''
...
```

bpftrace 是IO Visor开发的eBPF的追踪工具。它允许开发者用简洁的DSL（Domain Specific Language）编写eBPF程序，而不必在内核中手动编译和加载它们。具体验证可以参考 Brendan Gregg 的[A thorough introduction to bpftrace
](https://www.brendangregg.com/blog/2019-08-19/bpftrace.html)

## 在Kubernetes上调度 bpftrace 应用

Docker Desktop也内置了Kubernetes集群开发环境。为了帮助国内开发者解决由于无法访问 gcr.io 镜像仓库，导致无法开启 Kubernetes 的问题，阿里云团队维护了一个简单的 helper 工具可以解决相应问题，请参考： [https://github.com/AliyunContainerService/k8s-for-docker-desktop](https://github.com/AliyunContainerService/k8s-for-docker-desktop?spm=a2c6h.12873639.article-detail.10.3e805798KLBkfr) 。

kubectl-trace 是可以让用户在Kubernetes集群中安排执行bpftrace程序的kubectl插件。它的架构图如下，当通过 `kubectl trace` 命令创建一个trace应用时，kubectl-trace插件会在Kubernetes集群上，通过API创建一个被称作“trace-runner”的临时任务来执行bpftrace应用，可以调度到指定的节点或者Pod对其进行追踪。

![image](/images/macebpf1.png)

kubectl-trace的安装过程比较简单，请参考 [Installing](https://github.com/iovisor/kubectl-trace?spm=a2c6h.12873639.article-detail.11.3e805798KLBkfr#installing) 进行操作。然而其默认带的 trace-runner 镜像是不支持 Docker Desktop，执行会自动退出。我们来根据上文原理，hack 一个扩展的版本。其中的代码实现在 [https://github.com/denverdino/trace-runner-for-docker-desktop](https://github.com/denverdino/trace-runner-for-docker-desktop?spm=a2c6h.12873639.article-detail.12.3e805798KLBkfr) 项目中。

其实现与上文类似，在 /usr/src/ 中添加内核源代码，并挂载 debugfs , 然后就可以愉快的玩耍了。具体实现可以参见项目中 Dockerfile 与 main.go ，此处不在赘述。其运行效果如下：

```bash
$ kubectl trace run docker-desktop --imagename=registry.cn-hangzhou.aliyuncs.com/denverdino/kubectl-trace-runner -e "tracepoint:syscalls:sys_enter_* { @[probe] = count(); }"
trace 7b64f4b4-226e-4aaf-83b1-94858c72f91f created

$ kubectl trace attach 7b64f4b4-226e-4aaf-83b1-94858c72f91f
Attaching 327 probes...

^C
first SIGINT received, now if your program had maps and did not free them it should print them out

@[tracepoint:syscalls:sys_enter_getrlimit]: 1
@[tracepoint:syscalls:sys_enter_newstat]: 1
@[tracepoint:syscalls:sys_enter_fsync]: 1
@[tracepoint:syscalls:sys_enter_rt_sigsuspend]: 2
...
```

或者利用别名来简化命令使用

```bash
$ alias kubectl-trace-run="kubectl trace run --imagename=registry.cn-hangzhou.aliyuncs.com/denverdino/kubectl-trace-runner"
$ kubectl-trace-run docker-desktop -e 'tracepoint:syscalls:sys_enter_open { printf("%s %s\n", comm, str(args->filename)); }'
trace 96209723-f439-4fb8-8bdc-d32e41e53e35 created

$ kubectl trace attach 96209723-f439-4fb8-8bdc-d32e41e53e35
Attaching 1 probe...
sntpc /etc/services
sntpc /dev/urandom
sntpc /dev/urandom
...
```

Have fun and Happy Halloween!

本文引自[这里](https://developer.aliyun.com/article/798714)

