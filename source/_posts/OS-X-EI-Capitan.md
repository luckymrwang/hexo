title: OS X EI Capitan
date: 2015-10-04 14:01:05
tags: [OSX]
---

OS X El Capitan中，在内核下引入了Rootless机制，以下路径：

/System

/bin

/sbin

/usr (except /usr/local)

均属于Rootless范围，即使root用户无法对此目录有写和执行权限，只有Apple以及Apple授权签名的软件（包括命令行工具）可以修改此目录。

关闭Rootless(需要重启，以Recovery分区启动，在Security Configuration中关闭Enforce System Integrity Protection)