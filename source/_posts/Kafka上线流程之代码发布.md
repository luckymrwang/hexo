title: "Kafka上线流程之代码发布"
date: 2015-06-05 21:14:06
tags: [Kafka]
---

- kafka机器资源申请

- kafka开源软件部署，具体可以[查看这里]()

- kafka机器ID标识配置。每台机器的标识为我们BI里分配的大平台ID。例如坦克风云港台的大平台ID为1013。将此ID配置到港台kafka机器的/etc/rayjoy_plattech/machine.conf文件里：
<!-- more -->
		big_app_id = 1013

- 相关代码里增加这台机器的IP等配置

	1. bi-kafka-consumer-go项目里的conf/machine.conf里需要增加配置。在每个平台自己的section里增加kafka的brokerList配置，如：

			[1013]
			brokerList = 192.168.11.231:9092

	2. bi里需要配置该平台bi-kafka-consumer-go的url，如：

			$config['kafka_game_config'][1013] = array(
    			'msg_url' => 'http://tankkafka.efuntw.com:65400/',
			);

- 发布代码及启动我们的程序。需要登陆代码发布服务器，将bi-kafka-consumer-go发布到这台新kafka机器上