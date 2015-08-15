title: "Kafka开源软件配置及启动"
date: 2015-06-05 21:25:55
tags: [Kafka]
---

*以下安装均在root权限下操作*

	shell> sudo su -

- 安装kafka开源软件

		shell> mkdir -p /data/plattech
		shell> cd /data/plattech
		shell> wget http://mirrors.cnnic.cn/apache/kafka/0.8.1.1/kafka_2.9.2-0.8.1.1.tgz
		shell> tar -xzvf kafka_2.9.2-0.8.1.1.tgz
		shell> cd kafka_2.9.2-0.8.1.1
<!-- more -->
- 安装kafka运行环境java

		shell> yum install java-1.8.0-openjdk

- 配置系统hosts文件

		shell> vim /etc/hosts
*把机器的主机名（hostname）加到系统hosts文件后面*

##kafka-zookeeper

###配置

- zookeeper.properties

	`dataDir=/data/kafka-data/zookeeper（数据地址）`

	`clientPort=2181（kafka-zookeeper的port；一般用2181这个默认端口即可）`

###启动

	nohup bin/zookeeper-server-start.sh config/zookeeper.properties >zookeeper.nohup &

##kafka-broker

###配置

- server.properties

	`host.name=（本机局域网IP，注意不要配成127.0.0.1或localhost了，否则局域网其他机器不能访问）（有些机器需要配置成外网IP）`

	`num.partitions=1（分区数量，咱们暂时都是1）`

	`log.retention.hours=（日志保留时间，根据平台而定：半个月为360，2个月为1440。目前只有港台因为法律规定要2个月，其他均为半个月）`

	`log.dirs=/data/kafka-data/kafka-logs`

	`zookeeper.connect=localhost:2181（kafka-zookeeper的port；看情况是否需要重配）`

	`port=9092（broker自己的port；一般用默认的9092即可）`

	`num.network.threads=（看情况，一般用默认即可）`

	`num.io.threads=（看情况，一般用默认即可）`

###启动

	nohup bin/kafka-server-start.sh config/server.properties >server.nohup &

##测试

###第一次安装的测试

	bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

	bin/kafka-topics.sh --list --zookeeper localhost:2181

*显示test*

	bin/kafka-console-producer.sh --broker-list (LAN-ip,not127.*):9092 --topic test

*生产点消息，例如输入hello plattech，回车，ctrl-c结束程序*

	bin/kafka-console-consumer.sh --zookeeper (LAN-ip,not 127.*):2181 --topic test --from-beginning --property print.key=true

*会显示刚才生产的消息，如上的则是显示hello plattech*

###以后每次重启服务的测试

执行上边的第4步骤即可

##游戏是否生产数据的测试（需要等go的kafka consumer部署之后）

	http://tank-360-kafka.raysns.com:65400/v1/query/get_offset?type=last

##亚马逊云特殊情况

	yum install java装的是1.6的，会报内存错误

*要这么装*

	sudo yum install java-1.8.0-openjdk

##其他情况

运行bin/kafka-server-start.sh后找不到domain，需要将本机本地domain加到/etc/hosts里

##出错处理

如果在第三步测试的时候出现“SLF4J”错误时，说明kafka的libs库里少了一个jar包

解决办法：

1. 下载slf4j-1.7.x.zip。下载地址：http://www.slf4j.org/dist/slf4j-1.7.x.zip
  (注意版本要和kafka的libs库里的slf4j-api-1.7.x.jar相同)

2. 解压

	shell> unzip slf4j-1.7.x.zip

3. 把slf4j-nop-1.7.x.jar包复制到kafka的libs目录下面

	shell> cp -p ./slf4j-nop-1.7.x.jar /data/plattech/kafka_2.9.2-0.8.1.1/libs

