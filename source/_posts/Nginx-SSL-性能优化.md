title: Nginx SSL 性能优化
date: 2015-10-09 15:18:03
tags: [Nginx]
---

### 影响HTTPS速度的主要原因：

#### 密钥交换算法
常见的密钥交换算法有 `RSA`，`ECDHE`，`DH`，`DHE` 等算法。它们的特性如下：

	RSA：算法实现简单，诞生于 1977 年，历史悠久，经过了长时间的破解测试，安全性高。缺点就是需要比较大的素数（目前常用的是 2048 位）来保证安全强度，很消耗 CPU 运算资源。RSA 是目前唯一一个既能用于密钥交换又能用于证书签名的算法。
	
	DH：diffie-hellman 密钥交换算法，诞生时间比较早（1977 年），但是 1999 年才公开。缺点是比较消耗 CPU 性能。
	
	ECDHE：使用椭圆曲线（ECC）的 DH 算法，优点是能用较小的素数（256 位）实现 RSA 相同的安全等级。缺点是算法实现复杂，用于密钥交换的历史不长，没有经过长时间的安全攻击测试。
	
	ECDH：不支持 PFS，安全性低，同时无法实现 false start。
	
	DHE：不支持 ECC。非常消耗 CPU 资源 。

<!-- more -->
建议优先支持 RSA 和 ECDH_RSA 密钥交换算法。原因是：

1，  ECDHE 支持 ECC 加速，计算速度更快。支持 PFS，更加安全。支持 false start，用户访问速度更快。

2，  目前还有至少 20% 以上的客户端不支持 ECDHE，我们推荐使用 RSA 而不是 DH 或者 DHE，因为 DH 系列算法非常消耗 CPU（相当于要做两次 RSA 计算）。


更改其配置如下（参照 Nginx Performance Tuning for SSL（ http://dojo.techsamurais.com/?p=1384 ）：

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers ECDHE-RSA-AES256-SHA384:AES256-SHA256:RC4:HIGH:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!AESGCM;
	ssl_prefer_server_ciphers on;
	ssl_session_cache shared:SSL:10m;
	ssl_session_timeout 10m;

或者

	ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-RC4-SHA:!ECDHE-RSA-RC4-SHA:ECDH-ECDSA-RC4-SHA:ECDH-RSA-RC4-SHA:ECDHE-RSA-AES256-SHA:!RC4-SHA:HIGH:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!CBC:!EDH:!kEDH:!PSK:!SRP:!kECDH;

*（禁用了RC4，sha1，MD5等算法）*

#### 辅助加速：

##### 1. 启用SPDY

SPDY 是 Google 推出的优化 HTTP 传输效率的协议（ https://www.chromium.org/spdy ）, 它基本上沿用了 HTTP 协议的语义, 但是通过使用帧控制实现了多个特性，显著提升了 HTTP 协议的传输效率。 
SPDY 最大的特性就是多路复用，能将多个 HTTP 请求在同一个连接上一起发出去，不像目前的 HTTP 协议一样，只能串行地逐个发送请求。
可以 在编译Nginx带上参数 --with-http_spdy_module 支持SPDY协议，然后可在配置中启用： 

	listen 443 ssl spdy;

*检测是否使用SPDY的网址（ https://spdycheck.org/ ）*

##### 2. HSTS

HSTS(HTTP Strict Transport Security)。服务端返回一个 HSTS 的 http header，浏览器获取到 HSTS 头部之后，在一段时间内，不管用户输入 www.baidu.com 还是 http://www.baidu.com ，都会默认将请求内部跳转成 https://www.baidu.com；
将下述行添加到你的 HTTPS 配置的 server 块中：

	add_header Strict-Transport-Security "max-age=31536000";

##### 3. Session cache

Session cache 的原理是使用 client hello 中的 session id 查询服务端的 session cache, 如果服务端有对应的缓存，则直接使用已有的 session 信息提前完成握手，称为简化握手。

Session cache 有两个缺点：

1，    需要消耗服务端内存来存储 session 内容。

2，    目前的开源软件包括 nginx,apache 只支持单机多进程间共享缓存，不支持多机间分布式缓存，对于百度或者其他大型互联网公司而言，单机 session cache 几乎没有作用。
Session cache 也有一个非常大的优点：session id 是 TLS 协议的标准字段，市面上的浏览器全部都支持 session cache。

ssl_session_cache    shared:SSL:20m;
ssl_session_timeout  20m;
参照Nginx的官方文档1MB内存大约可以存储4000个session，按例配置20M大约可以存储80000。根据需求合理设置

4. Ocsp stapling
Ocsp 全称在线证书状态检查协议 (rfc6960)，用来向 CA 站点查询证书状态，比如是否撤销。通常情况下，浏览器使用 OCSP 协议发起查询请求，CA 返回证书状态内容，然后浏览器接受证书是否可信的状态。 将证书保存下来，浏览器请求时候直接通过自己的服务器发送回去，防止验证服务器出问题，还能加快访问速度。

查看OSCP验证服务器地址：

	openssl x509 -in 1_test.qupeiyin.net_bundle.crt  -text

在输出的文字中找到  OCSP - URI: ，后面的 URL 就是 OSCP 的验证服务器地址。
如图：


  请求OSCP证书
  
	openssl ocsp -noverify \             
	-issuer /certificate-path/trustchain.crt \             
	-cert /certificate-path/trustchain.crt \             
	-url http://ocsp1.wosign.com/ca6/server1

不出意外会收到如下的结果

     trustchain.crt: good        
         This Update: Oct 18 17:59:10 2014 GMT        
         Next Update: Oct 18 23:59:10 2014 GMT

如果出现 403 错误，那就需要在 Header 请求头加上域名参数 -header "HOST" "ocsp2.globalsign.com" ，没问题后就可以直接保存下来证书文件，完整的命令如下：

	openssl
	ocsp -noverify  -issuer 1_root_bundle.crt
	-cert 1_root_bundle.crt -url http://ocsp1.wosign.com/ca6/server1  -header "HOST"
	"ocsp1.wosign.com" -text -respout ./stapling_file.ocsp


将保存下来的 stapling_file.ocsp 证书添加到 nginx 的配置中，如下，Nginx 中配置变成了这样子：

    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_stapling_file /stapling_file.ocsp;
    ssl_trusted_certificate /certificate-path/trustchain.crt;

这样子重启 Nginx 后就会生效，可以使用下面的命令测试生效结果：

	echo QUIT | openssl s_client -connect blog.alphatr.com:443 -status 2> /dev/null | grep -A 17 'OCSP response:' | grep -B 17 'Next Update'

看到 OCSP Response Status: successful 这样的字样就是成功了。
ocsp证书有效期很短，大概不到一个月，所以过段时间要更新ocsp证书，不然还是会验证失败。需要用脚本定时更新OCSP证书。

### 总结：（全部优化参数）

	ssl on;
	ssl_certificate /data/www/ssl/ssl.crt;
	ssl_certificate_key /data/www/ssl/ssl.key;
	ssl_trusted_certificate /data/www/ssl/trustchain.crt;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers ECDHE-RSA-AES256-SHA384:AES256-SHA256:RC4:HIGH:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!AESGCM;
	ssl_prefer_server_ciphers on;
	ssl_stapling on;
	ssl_stapling_verify on;
	ssl_session_cache shared:SSL:10m;
	ssl_session_timeout 10m;
	add_header Strict-Transport-Security "max-age=31536000";
	resolver 223.5.5.5 223.6.6.6 valid=300s;
	resolver_timeout 10s;

	error_page 497 https://$host$request_uri;

#### 参数详解：

	ssl on  开启SSL
	ssl_certificate  对应单张证书
	ssl_certificate_key  对应私钥
	ssl_trusted_certificate  对应信任链(即Startcom SSL或者Positive SSL中需要附加到单张证书后面的那两张证书，可以独立出来)
	ssl_protocols  支持的SSL协议标准（nginx默认参数为：ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;）
	ssl_ciphers  （nginx默认参数为：ssl_ciphers HIGH:!aNULL:!MD5;）

	ssl_prefer_server_ciphers On; #指定服务器密码算法在优先于客户端密码算法时，使用SSLv3和TLS协议。

	error_page 497 https://$host$request_uri;  通过497错误将http转跳到https