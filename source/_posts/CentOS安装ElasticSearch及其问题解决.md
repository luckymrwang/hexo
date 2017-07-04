title: CentOS安装ElasticSearch及其问题解决
date: 2017-07-04 12:24:28
tags: [ES,CentOS]
---
进入 elasticsearch 的 bin 目录，尝试使用 ./elasticsearch 命令启动 elasticsearch 提示如下错误信息：

	Exception in thread "main" java.lang.RuntimeException: don't run elasticsearch as root.
	
解决方法：

1、指定允许root启动。如：
	 ./elasticsearch -Des.insecure.allow.root=true
	 
 可以启动，但会提示
 	[WARN ][bootstrap] running as ROOT user. this is a bad idea!
 	
2、创建一个非root elasticsearch相关的账号

- 创建一个分组，取名为esgroup，然后，往该分组中添加用户es，并设置es账户的密码
- 修改elasticsearch目录权限
- 使用新创建的账号es来登录终端启动

使用 `curl -X GET http://localhost:9200` 查看信息


推荐使用方法二启动，另外如果要在浏览器中查看，需要修改 elasticsearch.yml 中的 `network.host: 127.0.0.1` 为 `network.host: 0.0.0.0`