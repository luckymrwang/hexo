title: "Kafka代码自动化部署"
date: 2015-06-05 21:41:42
tags: [Kafka]
---

## 概述

kafka代码自动化部署的操作在deploy机器上。当前整个deploy比较简单，还需要登入deploy机器才能操作，以后有时间会慢慢改进成web方式直接操作。

## 辅助查看deploy机器已经配置了哪些kafka机器的信息

在http://203.195.156.75/deploy/里登陆后，在配置总览→查看ansible配置文件，这个页面其实就是现实ansible配置文件里的东西。
<!-- more -->
## 添加一个kafka机器信息到deploy机器的ansible配置文件里(root权限)

ansible的配置文件在/etc/ansible/hosts文件里

	[kafka]
	kafka_1009 ansible_ssh_host=203.195.131.211 ansible_ssh_port=10220 ansible_ssh_user=rayjoy
	[devsdk]
	devsdk ansible_ssh_host=119.28.5.63 ansible_ssh_port=10220 ansible_ssh_user=rayjoy

其中[kafka]这个section里配置都是kafka机器的，其他每个section对应一类机器。例如kafka_1009的配置，第一列为标记这台kafka机器对应的大平台，用的标识就是该太机器服务的大平台的统计ID。后边的host、port、user分别对应那台kafka机器的ip、ssh端口、和ssh登陆的用户

## 将deploy机器的ssh key加到目标kafka机器的信任里（rayjoy权限）

由于ansible走的ssh协议，我们现在用ssh key方式来进行deploy和kafka机器的通信比较便捷。

1. 拷贝deploy机器的ssh key，在文件~/.ssh/id_rsa.pub里。（！注意：即使为了方便，不用每次加都去deploy拷key，也不能把这个key写到公共文档里，可以写到自己私人笔记里）
2. 将这个key放到目标kafka机器的~/.ssh/authorized_keys文件最后一行即可。

## 向某台kafka机器更新发布bi-kafka-consumer-go项目可执行程序（rayjoy权限）

1. 登陆到deploy这台机器
2. 进入bi-kafka-consumer-go项目的代码目录的发布脚本文件夹

	shell> cd /home/rayjoy/plattech/go/src/rayjoy.com/plattech/bi-kafka-consumer-go/deploy-scripts

3. 将deploy-scripts/ansible.sh文件里的ansible_group的值改为你要发布代码的目标kafka机器标识，例如要发布代码到1009平台的kafka机器上，则修改为

	ansible_group=kafka_1009

4. 执行自动发布脚本，该脚本会自动更新、编译、部署

	shell> sh ansible.sh

5. ansible.sh会执行的操作

    a. 拉取更新最新的develop分支代码

    b. 编译bi-kafka-consumer-go这个项目的代码为二进制可执行程序

    c. 利用ansible将可执行程序、程序配置、重启脚本等同步到目标kafka机器

    d. 利用ansible远程执行该二进制可执行程序的重启脚本

    e. 最终目标机器就可以跑新版本的可执行程序了


