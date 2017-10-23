title: Supervisor安装使用
date: 2016-12-23 12:35:26
tags: [Linus,Centos]
toc: true
---

### 简介

`Supervisor` 是一个 `Python` 开发的 `client/server` 系统，可以管理和监控类 `UNIX` 操作系统上面的进程。它可以同时启动，关闭多个进程，使用起来特别的方便。

### 组成部分

supervisor 主要由两部分组成：

- supervisord(server 部分)：主要负责管理子进程，响应客户端命令以及日志的输出等；
- supervisorctl(client 部分)：命令行客户端，用户可以通过它与不同的 supervisord 进程联系，获取子进程的状态等。

<!-- more -->
### 安装

	sudo pip install supervisor
	
安装完成之后，可以运行 `echo_supervisord_conf` 生成默认的配置文件：

	echo_supervisord_conf > /etc/supervisord.conf

*我们把上面这部分配置保存到 /etc/supervisord.conf（或其他任意有权限访问的文件），然后启动 supervisord（通过 -c 选项指定配置文件路径，如果不指定会按照这个顺序查找配置文件：$CWD/supervisord.conf, $CWD/etc/supervisord.conf, /etc/supervisord.conf）*

### 运行

通过如下命令启动：

	supervisord -c /etc/supervisord.conf
	
### 进入 supervisorctl 的 shell 界面

*Supervisorctl 是 supervisord 的一个命令行客户端工具，启动时需要指定与 supervisord 使用同一份配置文件，否则与 supervisord 一样按照顺序查找配置文件。*

```
$ supervisorctl -c /etc/supervisord.conf
supervisor> status
supervisor>
```

由于目前没有添加任何需要管理的进程，所以 status 没有输出任何结果，接下来我们添加一个需要管理的进程 (以启动一个 celery 的 worker 为例)：

```
[program:notify]
directory=/data/go/src/celeryd ;
command=/data/go/src/celeryd/notify ;
stdout_logfile=/var/log/supervisor/notify_out.log ; stdout 日志输出位置
stderr_logfile=/var/log/supervisor/notify_err.log ; stderr 日志输出位置
autostart=true ; 在 supervisord 启动的时候自动启动
autorestart=true ; 程序异常退出后自动重启
startsecs=10 ; 启动 10 秒后没有异常退出，就当作已经正常启动
```
*一份配置文件至少需要一个 [program:x] 部分的配置，来告诉 supervisord 需要管理那个进程。[program:x] 语法中的 x 表示 program name，会在客户端（supervisorctl 或 web 界面）显示，在 supervisorctl 中通过这个值来对程序进行 start、restart、stop 等操作。*

然后运行以下命令更新配置并启动进程：

```
$ supervisorctl reread (只更新配置文件)
celeryd: available

$ supervisorctl update (只启动有改动的进程)
celeryd: added process group

$ supervisorctl status
celeryd                          RUNNING    pid 1919, uptime 0:00:18
```

我们看到 celery worker 已经被成功启动了。你可以使用不同的命令来控制进程的启动和关闭。

```
$ supervisorctl stop celeryd
celeryd: stopped
$ supervisorctl start celeryd
celeryd: started
$ supervisorctl restart celeryd
celeryd: stopped
celeryd: started
```

把所有的配置文件都放在 supervisord.conf 并不是个好主意，一旦管理的进程过多，就很麻烦。所以一般都会 新建一个目录来专门放置进程的配置文件，然后通过 include 的方式来获取这些配置信息

```
[include]
files = /etc/supervisor.d/*.ini ; 可以是 *.conf 或 *.ini
```

然后在目录 /etc/supervisor.d 下新建一个配置文件 celery.conf, 配置信息与上面的一致，效果是一样的。

### 命令详解

- supervisord: 初始启动Supervisord，启动、管理配置中设置的进程
- supervisorctl stop(start, restart) xxx，停止（启动，重启）某一个进程(xxx)
- supervisorctl reread: 只载入最新的配置文件, 并不重启任何进程
- supervisorctl reload: 载入最新的配置文件，停止原来的所有进程并按新的配置启动管理所有进程
- supervisorctl update: 根据最新的配置文件，启动新配置或有改动的进程，配置没有改动的进程不会受影响而重启

本文引自[这里](http://liuzxc.github.io/blog/supervisor/)



	
