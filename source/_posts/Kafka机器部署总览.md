title: "Kafka机器部署总览"
date: 2015-06-05 21:18:55
tags: [Kafka]
---

## kafka服务器将会部署的程序

### 日志kafka服务器程序及文件存储（平台技术部负责）

1. kafka的zookeeper、broker-server程序
2. kafka的文件存储
3. mysql少量存储
4. go编译后的http api程序bi-kafka-consumer-go

<!-- more -->
### 游戏滚服数据存储（坦克项目负责）

1. php程序
2. mysql存储

### 客服系统程序及存储（平台技术部负责）

1. php程序
2. mysql存储

## 综合后，基础软件

1. 此机器的域名
2. nginx 同tank游戏
3. php 同tank游戏
4. mysql 同tank游戏
5. kafka软件（http://mirrors.cnnic.cn/apache/kafka/0.8.1.1/kafka_2.9.2-0.8.1.1.tgz），这个为0.8.1.1版本
6. java（kafka依赖java启动），版本为1.7+即可

## 端口占用

1. kafka的zookeeper会占用2181，局域网内可访问即可
2. kafka的broker server会占用9092，局域网内可访问即可
3. bi-kafka-consumer-go会占用65400，需要外网访问

## 数据存储位置，请注意不要误删、误改

1. /etc/rayjoy_plattech/machine.conf，这个作为机器信息配置，目前kafka-consumer程序依赖这个做自适应配置
2. /home/rayjoy/plattech/airport，这个为我们的ansible自动上传文件的中转站，以及部分程序直接运行的位置
3. /data/kafka-data/，这个是kafka日志文件存储的地方
4. /data/plattech/，这个是我们平台技术部会放的一些程序位置，如当前kafka的开源软件会放这里，并从这里运行
5. mysql里game_kafka数据库，为我们bi-kafka-consumer程序所使用

## 代码自动发布及程序重启

1. 当前我们用的ansible的简易功能
2. bi-kafka-consumer的发布
	(1)并行上传到各机器的/home/rayjoy/plattech/airport/bi-kafka-consumer
	(2)并行将各机器的php再从/home/rayjoy/plattech/airport/bi-kafka-consumer同步到/opt/tankserver/game/webroot/bi-kafka-consumer
3. bi-kafka-consumer-go的发布和重启
	(1)并行上传到各机器的/home/rayjoy/plattech/airport/bi-kafka-consumer-go
	(2)并行运行个机器bi-kafka-consumer-go里的restart.sh脚本