title: PHP传参调试
date: 2015-10-05 12:39:44
tags: [调试]
---

通过调用系统环境变量$_SERVER，可以查看到HTTP请求的信息，如下：

~~~php
	$_SERVER["REMOTE_ADDR"]   //记录来访者的IP
	$_SERVER["QUERY_STRING"]  //查询请求字符串
~~~

<!-- more -->
把其加入代上面的代码中，并且将其写入到本地文件中来，全部代码如下：

~~~php
	private function traceHttp(){
		$this->logger("REMOTE_ADDR:".$_SERVER["REMOTE_ADDR"].((strpos($_SERVER["REMOTE_ADDR"], "101.226"))?" From Weixin":" Unknown IP"));
		$this->logger("QUERY_STRING:".$_SERVER["QUERY_STRING"]);
	}

	private function logger($contentStr){
		file_put_contents("log.html", date('Y-m-d H:i:s ').$contentStr."<br>", FILE_APPEND);
	}

~~~

以`CodeIgniter`为例，生成的文件与`index.php`同级目录