title: wrk - http性能测试工具使用指南
date: 2019-03-18 15:30:50
tags: [wrk]
---

[wrk](https://github.com/wg/wrk)

>wrk is a modern HTTP benchmarking tool capable of generating significant load when run on a single multi-core CPU. It combines a multithreaded design with scalable event notification systems such as epoll and kqueue.

>wrk使用了epoll(linux)和kqueue(mac)

<!-- more -->

### 安装

```js
brew install wrk
```

### 基本使用

1.使用帮助：

```js
Usage: wrk <options> <url>
  Options:
    -c, --connections <N>  Connections to keep open
    -d, --duration    <T>  Duration of test
    -t, --threads     <N>  Number of threads to use

    -s, --script      <S>  Load Lua script file
    -H, --header      <H>  Add header to request
        --latency          Print latency statistics
        --timeout     <T>  Socket/request timeout
    -v, --version          Print version details

  Numeric arguments may include a SI unit (1k, 1M, 1G)
  Time arguments may include a time unit (2s, 2m, 2h)
```

中文：

```js
使用方法: wrk <选项> <被测HTTP服务的URL>                            
  Options:                                            
    -c, --connections <N>  跟服务器建立并保持的TCP连接数量  
    -d, --duration    <T>  压测时间           
    -t, --threads     <N>  使用多少个线程进行压测   
                                                      
    -s, --script      <S>  指定Lua脚本路径       
    -H, --header      <H>  为每一个HTTP请求添加HTTP头      
        --latency          在压测结束后，打印延迟统计信息   
        --timeout     <T>  超时时间     
    -v, --version          打印正在使用的wrk的详细版本信息
                                                      
  <N>代表数字参数，支持国际单位 (1k, 1M, 1G)
  <T>代表时间参数，支持时间单位 (2s, 2m, 2h)
```

### 版本查看

```js
brew info wrk

wrk: stable 4.1.0 (bottled), HEAD
```

### 压测

```js
wrk -t12 -c400 -d30s --latency http://www.baidu.com

输出：
Running 30s test @ http://www.baidu.com（压测时间30s）
  12 threads and 400 connections（共12个测试线程，400个连接）
  Thread Stats   Avg      Stdev     Max   +/- Stdev
              （平均值） （标准差）（最大值）（正负一个标准差所占比例）
    Latency   933.14ms  415.00ms   2.00s    77.56%
    （延迟）
    Req/Sec    24.77     15.65   121.00     70.04%
    （处理中的请求数）
  Latency Distribution （延迟分布）
     50%  711.85ms
     75%    1.24s
     90%    1.56s
     99%    1.90s （99分位的延迟）
  8181 requests in 30.10s, 121.83MB read（30.10秒内共处理完成了8181个请求，读取了121.83MB数据）
  Socket errors: connect 0, read 0, write 0, timeout 1545
Requests/sec:    271.82 （平均每秒处理完成271.82个请求）
Transfer/sec:    4.05MB （平均每秒读取数据4.05MB）
```

以上使用12个线程400个连接，对baidu首页进行了30秒的压测，并要求在压测结果中输出响应延迟信息。

本文引自[这里](https://www.jianshu.com/p/28b613a6590c)


