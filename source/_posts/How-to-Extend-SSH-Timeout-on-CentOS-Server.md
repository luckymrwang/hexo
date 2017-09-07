title: How to Extend SSH Timeout on CentOS Server
date: 2016-08-02 12:03:23
tags: [Centos]
toc: true
---
用`vim`打开`/etc/ssh/sshd_config`, 定位到:

```
#ClientAliveInterval 0 
#ClientAliveCountMax 3
```
删掉前面的`#`号，并改变其值,例如:

```
ClientAliveInterval 120 
ClientAliveCountMax 10
```

`:wq`关闭并保存

*`120`表示客户端每隔120s将要给服务器发送一条数据。`10`表示客户端没有收到服务器的反馈后将会一直发送，直到达到该最大值停止*

重启`SSH`服务：

```
# service sshd restart 
Stopping sshd:            [ OK ] 
Starting sshd:            [ OK ]
```