title: Apache 个别参数解释
date: 2015-11-03 17:09:22
tags: [Apache]
toc: true
---

Apache可以直接使用2.0版本产品线。针对Apache的优化主要是针对httpd.conf的优化，当然还有其他地方，如果特别留意的话，网上常有专家惊呼“居然这么多人忽略xxxx处的优化”等等，实际情况也确实如此，因为优化的地方实在太多了，httpd.conf只能做一个出发点。即便如此如果仅仅使用httpd.conf出厂默认值的话还是令人痛心不已。

httpd.conf的优化点有以下几处：
<!-- more -->
#### KeepAlive

其实Akamai的图片存储服务主要解决服务器的KeepAlive问题。看下面这个sample.html：

	Hello world<img src="http://blog.penner.cn/hello.gif" />
	
当浏览器将请求发送给Apache后，Apache会为该用户建立连接，返回/sample.html的内容，浏览器解析HTML文件，发现还需要显示 /hello.gif，就再次向Apache发出请求。这时如果KeepAlive为Off，Apache就需要重新建立连接。试想如果页面请求了 1000个图片，Apache就需要建立1000个连接（但建立第N个时候N-x个连接已经被Apache聪明的关闭了)。如果KeepAlive为 On，Apache会在同一个连接中处理所有这些请求，大大的节省了连接资源，可惜这个世界上有很多攻击者，他们会利用这个连接不断的特性不停的请求文件，耗尽服务器的资源。所以一些大公司像Yahoo、AOL都选用Akamai作为图片存储服务，结果这些公司的sample.html版本就成了这个样子：

	Hello world<img src="/hello.gif" />
	
(真实地图片地址会比这个复杂)这样一来每次用户访问仅会向本机服务器的Apache请求一次，剩下的请求发送到akamai了。不必为akamai的能力担心，因为它有充足的抗负载技术。

#### MaxKeepAliveRequests

明白了1中的内容，这个看名字就知道一个连接可以最多发送多少次请求。默认是500。

#### KeepAliveTimeout

同样，两次请求间超过这个数字就中断这个连接。如果你的KeepAlive是On，MaxKeepAliveRequests是500， KeepAliveTimeout是100，你可以算算攻击者们用多久可以耗干你的Apache。我把KeepAliveTimeout设为5，因为从我网站受众人群的上网速度和网站的图片大小、数量考虑，5秒种可以完成加载多数页面。

#### StartServers

StartServers 的数字表示Apache启动后直接创建的httpd数量。比如你的服务器平时平均需要100个httpd，如果把StartServers设为10就会导致Apache启动之初不停的创建剩下的90个httpd。如果你的服务器平时最多就用20个httpd，把StartServers设为50就浪费了资源。这个参数没什么大不了，因为Apache会自己趋向于适合的httpd服务数。

#### MinSpareServers、MaxSpareServers

保留备用的httpd服务数最小值和最大值。即当不需要这么多httpd服务时，依然最少保留MinSpareServers个服务，但不超过MaxSpareServers个服务。需要根据Apache的运行寻找经验值。

#### ServerLimit，MaxClients

比较重要的一个值。ServerLimit通常应该等于MaxClients。MaxClients决定了最大的httpd进程数，如果攻击者占用了 MaxClients的httpd服务数，你的网站就拒绝正常访问者访问了。但MaxClients的大小受内存的限制，因此Apache2的默认值是 250，并加上了ServerLimit参数作限制，如果想设大MaxClients，必须同时扩大ServerLimit，但ServerLimit不应超过MaxClients。

#### MaxRequestsPerChild

决定了每个httpd服务可以处理的最大请求数，超过这个数字就需要新的httpd服务，后者又由MaxClients限制，环环相套。我的MaxRequestsPerChild是10000。

#### HostnameLookups

设为Off，避免DNS查询的等待。
除了这8个参数外，Apache的另一个可塑点是加载的Module，把不需要的LoadModule注释掉即可，大大的节省了内存。但是问题是你不知道那个Module不需要，即便对照着Apache的Module文档朗读各个Module作用，也只能注释掉很少几个。下面是注释掉的几个Module：
mime_magic_module、info_module、userdir_module、proxy_module、proxy_ftp_module、proxy_http_module、proxy_connect_module。