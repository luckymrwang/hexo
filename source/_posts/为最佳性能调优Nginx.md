title: 为最佳性能调优Nginx
date: 2015-08-21 16:26:58
tags: [Nginx]
---
通常来说，一个优化良好的 Nginx Linux 服务器可以达到 500,000 – 600,000 次/秒 的请求处理性能，然而我的 Nginx 服务器可以稳定地达到 904,000 次/秒 的处理性能，并且我以此高负载测试超过 12 小时，服务器工作稳定。

这里需要特别说明的是，本文中所有列出来的配置都是在我的测试环境验证的，而你需要根据你服务器的情况进行配置：
<!-- more -->

从 EPEL 源安装 Nginx：

```
yum -y install nginx
```

备份配置文件，然后根据你的需要进行配置：

```
    cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig
    vim /etc/nginx/nginx.conf
    # This number should be, at maximum, the number of CPU cores on your system.
    # (since nginx doesn't benefit from more than one worker per CPU.)
    # 这里的数值不能超过 CPU 的总核数，因为在单个核上部署超过 1 个 Nginx 服务进程并不起到提高性能的作用。
    worker_processes 24;
    
    # Number of file descriptors used for Nginx. This is set in the OS with 'ulimit -n 200000'
    # or using /etc/security/limits.conf
	# Nginx 最大可用文件描述符数量，同时需要配置操作系统的 "ulimit -n 200000"，或者在 /etc/security/limits.    conf 中配置。
    worker_rlimit_nofile 200000;
    
    # only log critical errors
    # 只记录 critical 级别的错误日志
    error_log /var/log/nginx/error.log crit
    
    # Determines how many clients will be served by each worker process.
    # (Max clients = worker_connections * worker_processes)
	# "Max clients" is also limited by the number of socket connections available on the system     (~64k)
    # 配置单个 Nginx 单个进程可服务的客户端数量，（最大值客户端数 = 单进程连接数 * 进程数 ）
    # 最大客户端数同时也受操作系统 socket 连接数的影响（最大 64K ）
    worker_connections 4000;
    
    # essential for linux, optmized to serve many clients with each thread
    # Linux 关键配置，允许单个线程处理多个客户端请求。
    use epoll;
    
	# Accept as many connections as possible, after nginx gets notification about a new     connection.
    # May flood worker_connections, if that option is set too low.
    # 允许尽可能地处理更多的连接数，如果 worker_connections 配置太低，会产生大量的无效连接请求。
    multi_accept on;
    
    # Caches information about open FDs, freqently accessed files.
	# Changing this setting, in my environment, brought performance up from 560k req/sec, to    904k req/sec.
	# I recommend using some varient of these options, though not the specific values listed    below.
    # 缓存高频操作文件的FDs（文件描述符/文件句柄）
    # 在我的设备环境中，通过修改以下配置，性能从 560k 请求/秒 提升到 904k 请求/秒。
    # 我建议你对以下配置尝试不同的组合，而不是直接使用这几个数据。
    open_file_cache max=200000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    
    # Buffer log writes to speed up IO, or disable them altogether
    # 将日志写入高速 IO 存储设备，或者直接关闭日志。
    # access_log /var/log/nginx/access.log main buffer=16k;
    access_log off;
    
    # Sendfile copies data between one FD and other from within the kernel.
	# More efficient than read() + write(), since the requires transferring data to and from the    user space.
    # 开启 sendfile 选项，使用内核的 FD 文件传输功能，这个比在用户态用 read() + write() 的方式更加高效。
    sendfile on;
    
    # Tcp_nopush causes nginx to attempt to send its HTTP response head in one packet,
	# instead of using partial frames. This is useful for prepending headers before calling     sendfile,
    # or for throughput optimization.
    # 打开 tcp_nopush 选项，Nginux 允许将 HTTP 应答首部与数据内容在同一个报文中发出。
    # 这个选项使服务器在 sendfile 时可以提前准备 HTTP 首部，能够达到优化吞吐的效果。
    tcp_nopush on;
    
	# don't buffer data-sends (disable Nagle algorithm). Good for sending frequent small bursts     of data in real time.
    # 不要缓存 data-sends （关闭 Nagle 算法），这个能够提高高频发送小数据报文的实时性。
    tcp_nodelay on;
    
    # Timeout for keep-alive connections. Server will close connections after this time.
    # 配置连接 keep-alive 超时时间，服务器将在超时之后关闭相应的连接。
    keepalive_timeout 30;
    
	# Number of requests a client can make over the keep-alive connection. This is set high for     testing.
    # 单个客户端在 keep-alive 连接上可以发送的请求数量，在测试环境中，需要配置个比较大的值。
    keepalive_requests 100000;
    
	# allow the server to close the connection after a client stops responding. Frees up socket-    associated memory.
    # 允许服务器在客户端停止发送应答之后关闭连接，以便释放连接相应的 socket 内存开销。
    reset_timedout_connection on;
    
    # send the client a "request timed out" if the body is not loaded by this time. Default 60.
    # 配置客户端数据请求超时时间，默认是 60 秒。
    client_body_timeout 10;
    
	# If the client stops reading data, free up the stale client connection after this much     time. Default 60.
    # 客户端数据读超时配置，客户端停止读取数据，超时时间后断开相应连接，默认是 60 秒。
    send_timeout 2;
    
    # Compression. Reduces the amount of data that needs to be transferred over the network
    # 压缩参数配置，减少在网络上所传输的数据量。
    gzip on;
    gzip_min_length 10240;
    gzip_proxied expired no-cache no-store private auth;
	gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/   xml;
    gzip_disable "MSIE [1-6].";
```

启动 Nginx 并配置起机自动加载。

```
	service nginx start
	chkconfig nginx on
```
	
配置 Tsung 并启动测试，测试差不多 10 分钟左右就能测试到服务器的峰值能力，具体的时间与你的 Tsung 配置相关。

```
	[root@loadnode1 ~] vim ~/.tsung/tsung.xml
      <server host="YOURWEBSERVER" port="80" type="tcp"/>
      
	tsung start
```
	
你觉得测试结果已经够了的情况下，通过 ctrl+c 退出，之后使用我们之前配置的别名命令 treport 查看测试报告。

WEB 服务器调优，第二部分：TCP 协议栈调优

这个部分不只是对 Ngiinx 适用，还可以在任何 WEB 服务器上使用。通过对内核 TCP 配置的优化可以提高服务器网络带宽。

以下配置在我的 10-Gbase-T 服务器上工作得非常完美，服务器从默认配置下的 8Gbps 带宽提升到 9.3Gbps。

当然，你的服务器上的结论可能不尽相同。

下面的配置项，我建议每次只修订其中一项，之后用网络性能测试工具 netperf、iperf 或是用我类似的测试脚本 cluster-netbench.pl 对服务器进行多次测试。

```
	yum -y install netperf iperf
	
	vim /etc/sysctl.conf

    # Increase system IP port limits to allow for more connections
    # 调高系统的 IP 以及端口数据限制，从可以接受更多的连接
    net.ipv4.ip_local_port_range = 2000 65000
    
    net.ipv4.tcp_window_scaling = 1
    
    # number of packets to keep in backlog before the kernel starts dropping them
    # 设置协议栈可以缓存的报文数阀值，超过阀值的报文将被内核丢弃
    net.ipv4.tcp_max_syn_backlog = 3240000
    
    # increase socket listen backlog
    # 调高 socket 侦听数阀值
    net.core.somaxconn = 3240000
    net.ipv4.tcp_max_tw_buckets = 1440000
    
    # Increase TCP buffer sizes
    # 调大 TCP 存储大小
    net.core.rmem_default = 8388608
    net.core.rmem_max = 16777216
    net.core.wmem_max = 16777216
    net.ipv4.tcp_rmem = 4096 87380 16777216
    net.ipv4.tcp_wmem = 4096 65536 16777216
    net.ipv4.tcp_congestion_control = cubic
```

每次修订配置之后都需要执行以下命令使之生效.
```
	sysctl -p /etc/sysctl.conf
```

别忘了在配置修订之后务必要进行网络 benchmark 测试，这样可以观测到具体是哪个配置修订的优化效果最明显。通过这种有效测试方法可以为你节省大量时间。