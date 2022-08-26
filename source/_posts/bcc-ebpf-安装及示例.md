title: bcc/ebpf 安装及示例
date: 2022-08-22 16:44:55
tags: [eBPF]
---
eBPF 程序使用 C 语言的一个子集（restricted C）编写，然后通过 LLVM 编译成字节码注入到 内核执行。bcc是 eBPF 的一个外围工具集，使得 “编 写 BPF 代码-编译成字节码-注入内核-获取结果-展示” 整个过程更加便捷。

下面我们将搭建一个基础环境，通过几个例子展示如何编写 bcc/eBPF 程序，感受它们的强大功能。

### 准备工作

环境需要以下几方面满足要求：内核、docker、bcc。

#### 内核版本

eBPF 需要较新的 Linux kernel 支持。 因此首先要确保你的内核版本足够新，至少要在 4.1 以上，最好在 4.10 以上：

```bash
$ uname -r
4.10.13-1.el7.elrepo.x86_64
```

<!-- more -->
#### docker

本文的示例需要使用 Docker，版本没有明确的限制，较新即可。

#### bcc 工具

bcc 是 python 封装的 eBPF 外围工具集，可以大大方面 BPF 程序的开发。

为方便使用，我们将把 bcc 打包成一个 docker 镜像，以容器的方式使用 bcc。打包镜像的过程 见附录 1，这里不再赘述。

下载 bcc 代码：

```sh
$ git clone https://github.com/iovisor/bcc.git
```

然后启动 bcc 容器：

```sh
$ cd bcc
$ sudo docker run -d --name bcc \
    --privileged \
    -v $(pwd):/bcc \
    -v /lib/modules:/lib/modules:ro \
    -v /usr/src:/usr/src:ro \
    -v /boot:/boot:ro \
    -v /sys/kernel/debug:/sys/kernel/debug \
    bcc:0.0.1 sleep infinity
```

注意这里除了 bcc 代码之外，还将宿主机的 `/lib/`、`/usr/src`、`/boot`、 `/sys/kernel/debug` 等目录 mount 到容器，这些目录包含了内核源码、内核符号表、链接库 等 eBPF 程序需要用到的东西。

#### 测试 bcc 工作正常

```sh
$ docker exec -it bcc bash
```

在容器内部执行 `funcslower.py` 脚本，捕获内核收包函数 `net_rx_action` 耗时大于 100us 的情况，并打印内核调用栈。注意，视机器的网络和工作负载状况，这里的打印可能没有，也可能会非常多。建议先设置一个比较大的阈值（例如`-u 200`），如果没有输出 ，再将阈值逐步改小。

```sh
root@container # cd /bcc/tools
root@container # ./funcslower.py -u 100 -f -K net_rx_action
Tracing function calls slower than 100 us... Ctrl+C to quit.
COMM           PID    LAT(us)             RVAL FUNC
swapper/1      0       158.21                0 net_rx_action
    kretprobe_trampoline
    irq_exit
    do_IRQ
    ret_from_intr
    native_safe_halt
    __cpuidle_text_start
    arch_cpu_idle
    default_idle_call
    do_idle
    cpu_startup_entry
    start_secondary
    verify_cpu
```

调节 `-u` 大小，如果有类似以上输出，就说明我们的 `bcc/eBPF` 环境可以用了。

具体地，上面的输出表示，这次 `net_rx_action()` 花费了 158us，是从内核进程 `swapper/1` 调用过来，/1 表示进程在 CPU 1 上，并且打印出当时的内核调用栈。通过这个简 单的例子，我们就隐约感受到了 `bcc/eBPF` 的强大。

### bcc/eBPF 程序示例

接下来我们通过编写一个简单的 eBPF 程序 `simple-biolatency` 来展示 bcc/eBPF 程序是如 何构成及如何工作的。

我们的程序会监听块设备 IO 相关的系统调用，统计 IO 操作的耗时（I/O latency）， 并打印出统计直方图。程序大致分为三个部分：

1. 核心 eBPF 代码 (hook)，C 编写，会被编译成字节码注入到内核，完成事件的采集和计时
2. 外围 Python 代码，完成 eBPF 代码的编译和注入
3. 命令行 Python 代码，完成命令行参数解析、运行程序、打印最终结果等工作

为方便起见，以上全部代码都放到同一个文件 `simple-biolatency.py`。

整个程序需要如下几个依赖库：

```python
from __future__ import print_function

import sys
from time import sleep, strftime

from bcc import BPF
```

#### BPF 程序

首先看 BPF 程序。这里主要做三件事情：

1. 初始化一个 BPF hash 变量 `start` 和直方图变量 `dist`，用于计算和保存统计信息
2. 定义 `trace_req_start()` 函数：在每个 I/O 请求开始之前会调用这个函数，记录一个时间戳
3. 定义 `trace_req_done()` 函数：在每个 I/O 请求完成之后会调用这个函数，再根据上一步记录的开始时间戳，计算出耗时

```c
bpf_text = """
#include <uapi/linux/ptrace.h>
#include <linux/blkdev.h>

BPF_HASH(start, struct request *);
BPF_HISTOGRAM(dist);

// time block I/O
int trace_req_start(struct pt_regs *ctx, struct request *req)
{
    u64 ts = bpf_ktime_get_ns();
    start.update(&req, &ts);
    return 0;
}

// output
int trace_req_done(struct pt_regs *ctx, struct request *req)
{
    u64 *tsp, delta;

    // fetch timestamp and calculate delta
    tsp = start.lookup(&req);
    if (tsp == 0) {
        return 0;   // missed issue
    }
    delta = bpf_ktime_get_ns() - *tsp;
    delta /= 1000;

    // store as histogram
    dist.increment(bpf_log2l(delta));

    start.delete(&req);
    return 0;
}
"""
```

#### 加载 BPF 程序

加载 BPF 程序，然后将 hook 函数分别插入到如下几个系统调用前后：

1. `blk_start_request`
2. `blk_mq_start_request`
3. `blk_account_io_done`

```python
b = BPF(text=bpf_text)
if BPF.get_kprobe_functions(b'blk_start_request'):
    b.attach_kprobe(event="blk_start_request", fn_name="trace_req_start")
b.attach_kprobe(event="blk_mq_start_request", fn_name="trace_req_start")
b.attach_kprobe(event="blk_account_io_done", fn_name="trace_req_done")
```

#### 命令行解析

最后是命令行参数解析等工作。根据指定的采集间隔（秒）和采集次数运行。程序结束的时 候，打印耗时直方图：

```python
if len(sys.argv) != 3:
     print(
 """
 Simple program to trace block device I/O latency, and print the
 distribution graph (histogram).

 Usage: %s [interval] [count]

 interval - recording period (seconds)
 count - how many times to record

 Example: print 1 second summaries, 10 times
 $ %s 1 10
 """ % (sys.argv[0], sys.argv[0]))
     sys.exit(1)

 interval = int(sys.argv[1])
 countdown = int(sys.argv[2])
 print("Tracing block device I/O... Hit Ctrl-C to end.")

 exiting = 0 if interval else 1
 dist = b.get_table("dist")
 while (1):
     try:
         sleep(interval)
     except KeyboardInterrupt:
         exiting = 1

     print()
     print("%-8s\n" % strftime("%H:%M:%S"), end="")

     dist.print_log2_hist("usecs", "disk")
     dist.clear()

     countdown -= 1
     if exiting or countdown == 0:
         exit()
```

#### 运行

实际运行效果：

```sh
root@container # ./simple-biolatency.py 1 2
Tracing block device I/O... Hit Ctrl-C to end.

13:12:21

13:12:22
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 0        |                                        |
      1024 -> 2047       : 0        |                                        |
      2048 -> 4095       : 0        |                                        |
      4096 -> 8191       : 0        |                                        |
      8192 -> 16383      : 12       |****************************************|
```

可以看到，第二秒采集到了 12 次请求，并且耗时都落在 `8192us ~ 16383us` 这个区间。

#### 小结

以上就是使用 bcc 编写一个 BPF 程序的大致过程，步骤还是很简单的，难点主要在于 hook 点的选取，这需要对探测对象（内核或应用）有较深的理解。实际上，以上代码是 bcc 自带的 `tools/biolatency.py` 的一个简化版，大家可以执行 `biolatency.py -h` 查看完整 版的功能。

### 更多示例

`bcc/tools` 目录下有大量和上面类似的工具，建议都尝试运行一下。这些程序通常都很短， 如果想自己写 bcc/BPF 程序的话，这是非常好的学习教材。

1. `argdist.py` 统计指定函数的调用次数、调用所带的参数等等信息，打印直方图
2. `bashreadline.py` 获取正在运行的 bash 命令所带的参数
3. `biolatency.py` 统计 block IO 请求的耗时，打印直方图
4. `biosnoop.py` 打印每次 block IO 请求的详细信息
5. `biotop.py` 打印每个进程的 block IO 详情
6. `bitesize.py` 分别打印每个进程的 IO 请求直方图
7. `bpflist.py` 打印当前系统正在运行哪些 BPF 程序
8. `btrfsslower.py` 打印 btrfs 慢于某一阈值的 read/write/open/fsync 操作的数量
9. `cachestat.py` 打印 Linux 页缓存 hit/miss 状况
10. `cachetop.py` 分别打印每个进程的页缓存状况
11. `capable.py` 跟踪到内核函数 cap_capable()（安全检查相关）的调用，打印详情
12. `ujobnew.sh` 跟踪内存对象分配事件，打印统计，对研究 GC 很有帮助
13. `cpudist.py` 统计 task on-CPU time，即任务在被调度走之前在 CPU 上执行的时间
14. `cpuunclaimed.py` 跟踪 CPU run queues length，打印 idle CPU (yet unclaimed by waiting threads) 百分比
15. `criticalstat.py` 跟踪涉及内核原子操作的事件，打印调用栈
16. `dbslower.py` 跟踪 MySQL 或 PostgreSQL 的慢查询
17. `dbstat.py` 打印 MySQL 或 PostgreSQL 的查询耗时直方图
18. `dcsnoop.py` 跟踪目录缓存（dcache）查询请求
19. `dcstat.py` 打印目录缓存（dcache）统计信息
20. `deadlock.py` 检查运行中的进行可能存在的死锁
21. `execsnoop.py` 跟踪新进程创建事件
22. `ext4dist.py` 跟踪 ext4 文件系统的 read/write/open/fsyncs 请求，打印耗时直方图
23. `ext4slower.py` 跟踪 ext4 慢请求
24. `filelife.py` 跟踪短寿命文件（跟踪期间创建然后删除）
25. `fileslower.py` 跟踪较慢的同步读写请求
26. `filetop.py` 打印文件读写排行榜（top），以及进程详细信息
27. `funccount.py` 跟踪指定函数的调用次数，支持正则表达式
28. `funclatency.py` 跟踪指定函数，打印耗时
29. `funcslower.py` 跟踪唤醒时间（function invocations）较慢的内核和用户函数
30. `gethostlatency.py` 跟踪 hostname 查询耗时
31. `hardirqs.py` 跟踪硬中断耗时
32. `inject.py`
33. `javacalls.sh`
34. `javaflow.sh`
35. `javagc.sh`
36. `javaobjnew.sh`
37. `javastat.sh`
38. `javathreads.sh`
39. `killsnoop.py` 跟踪 kill()系统调用发出的信号
40. `llcstat.py` 跟踪缓存引用和缓存命中率事件
41. `mdflush.py` 跟踪 md driver level 的 flush 事件
42. `memleak.py` 检查内存泄漏
43. `mountsnoop.py` 跟踪 mount 和 unmount 系统调用
44. `mysqld_qslower.py` 跟踪 MySQL 慢查询
45. `nfsdist.py` 打印 NFS read/write/open/getattr 耗时直方图
46. `nfsslower.py` 跟踪 NFS read/write/open/getattr 慢操作
47. `nodegc.sh` 跟踪高级语言（Java/Python/Ruby/Node/）的 GC 事件
48. `offcputime.py` 跟踪被阻塞的进程，打印调用栈、阻塞耗时等信息
49. `offwaketime.py` 跟踪被阻塞且 off-CPU 的进程
50. `oomkill.py` 跟踪 Linux out-of-memory (OOM) killer
51. `opensnoop.py` 跟踪 open()系统调用
52. `perlcalls.sh`
53. `perlstat.sh`
54. `phpcalls.sh`
55. `phpflow.sh`
56. `phpstat.sh`
57. `pidpersec.py` 跟踪每分钟新创建的进程数量（通过跟踪 fork()）
58. `profile.py` CPU profiler
59. `pythoncalls.sh`
60. `pythoonflow.sh`
61. `pythongc.sh`
62. `pythonstat.sh`
63. `reset-trace`.sh
64. `rubycalls.sh`
65. `rubygc.sh`
66. `rubyobjnew.sh`
67. `runqlat.py` 调度器 run queue latency 直方图，每个 task 等待 CPU 的时间
68. `runqlen.py` 调度器 run queue 使用百分比
69. `runqslower.py` 跟踪调度延迟很大的进程（等待被执行但是没有空闲 CPU）
70. `shmsnoop.py` 跟踪 shm*()系统调用
71. `slabratetop.py` 跟踪内核内存分配缓存（SLAB 或 SLUB）
72. `sofdsnoop.py` 跟踪 unix socket 文件描述符（FD）
73. `softirqs.py` 跟踪软中断
74. `solisten.py` 跟踪内核 TCP listen 事件
75. `sslsniff.py` 跟踪 OpenSSL/GnuTLS/NSS 的 write/send 和 read/recv 函数
76. `stackcount.py` 跟踪函数和调用栈
77. `statsnoop.py` 跟踪 stat()系统调用
78. `syncsnoop.py` 跟踪 sync()系统调用
79. `syscount.py` 跟踪各系统调用次数
80. `tclcalls.sh`
81. `tclflow.sh`
82. `tclobjnew.sh`
83. `tclstat.sh`
84. `tcpaccept.py` 跟踪内核接受 TCP 连接的事件
85. `tcpconnect.py` 跟踪内核建立 TCP 连接的事件
86. `tcpconnlat.py` 跟踪建立 TCP 连接比较慢的事件，打印进程、IP、端口等详细信息
87. `tcpdrop.py` 跟踪内核 drop TCP 包或片（segment）的事件
88. `tcplife.py` 打印跟踪期间建立和关闭的的 TCP session
89. `tcpretrans.py` 跟踪 TCP 重传
90. `tcpstates.py` 跟踪 TCP 状态变化，包括每个状态的时长
91. `tcpsubnet.py` 根据 destination 打印每个 subnet 的 throughput
92. `tcptop.py` 根据 host 和 port 打印 throughput
93. `tcptracer.py` 跟踪进行 TCP connection 操作的内核函数
94. `tplist.py` 打印内核 tracepoint 和 USDT probes 点，已经它们的参数
95. `trace.py` 跟踪指定的函数，并按照指定的格式打印函数当时的参数值
96. `ttysnoop.py` 跟踪指定的 tty 或 pts 设备，将其打印复制一份输出
97. `vfscount.py` 统计 VFS（虚拟文件系统）调用
98. `vfsstat.py` 跟踪一些重要的 VFS 函数，打印统计信息
99. `wakeuptime.py` 打印进程被唤醒的延迟及其调用栈
100. `xfsdist.py` 打印 XFS read/write/open/fsync 耗时直方图
101. `xfsslower.py` 打印 XFS 慢请求
102. `zfsdist.py` 打印 ZFS read/write/open/fsync 耗时直方图
103. `zfsslower.py` 打印 ZFS 慢请求

### References

1. [Kernel Document: A thorough introduction to eBPF](https://lwn.net/Articles/740157/)
2. [bcc: Install Guide](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

### 附录 1：打包 bcc 镜像

本节描述如何基于 ubuntu 18.04 打包一个 bcc 镜像，内容参考自 [bcc 官方编译教程](https://github.com/iovisor/bcc/blob/master/INSTALL.md)。

首先下载 ubuntu:20.04 作为基础镜像：

```sh
$ docker pull ubuntu:20.04
```

然后将如下内容保存为 `Dockerfile`：

```sh
FROM ubuntu:20.04

RUN apt update && apt install -y gnupg lsb-core
RUN apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4052245BD4284CDD
RUN echo "deb https://repo.iovisor.org/apt/$(lsb_release -cs) $(lsb_release -cs) main" > tee /etc/apt/sources.list.d/iovisor.list
RUN apt-get install bcc-tools libbcc-examples linux-headers-$(uname -r)
```

生成镜像：

```sh
$ sudo docker build -t bcc:0.0.1
```

附录 2：基于构建好的[镜像](https://luckymrwang.github.io/2022/05/23/%E4%BD%BF%E7%94%A8-Docker-Desktop%E8%BF%9B%E8%A1%8C-BPF-%E5%BC%80%E5%8F%91/)

```sh
docker run -d --name bcc \
    --privileged \
    -v /lib/modules:/lib/modules:ro \
    -v /usr/src:/usr/src:ro \
    -v /boot:/boot:ro \
    -v /etc/localtime:/etc/localtime:ro \
    --pid=host \
    --workdir /root/bcc/tools \
    registry.cn-hangzhou.aliyuncs.com/denverdino/ebpf-for-mac sleep infinity
```




