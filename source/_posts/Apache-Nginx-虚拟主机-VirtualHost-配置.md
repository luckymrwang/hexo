title: "Apache&Nginx 虚拟主机 VirtualHost 配置"
date: 2015-06-03 14:35:04
tags: [Apache,Nginx]
---

##概念

**虚拟主机 (Virtual Host)** 是在同一台机器搭建属于不同域名或者基于不同 IP 的多个网站服务的技术. 可以为运行在同一物理机器上的各个网站指配不同的 IP 和端口, 也可让多个网站拥有不同的域名。

*重要：Apache 在接受到请求时，首先会默认第一个VirtualHost，然后再找匹配的，如果没有匹配的，就是第一个VirtualHost起作用。因此在httpd.conf中，将<Dicrectory />（这个是所有目录的默认配置)和<Direcotry /opt/lampp/htdocs>的权限，都是deny from all.作为默认。*
<!-- more -->
##Redhat Enterprise Linux

Redhat Enterprise Linux (包括 CentOS Linux), 是使用最广的 Linux 服务器, 大量的网站应用都部署在其上。

- 打开文件 /etc/httpd/conf/httpd.conf, 搜索 VirtualHost example, 找到代码如下:

		# VirtualHost example:
		# Almost any Apache directive may go into a VirtualHost container.
		# The first VirtualHost section is used for requests without a known
		# server name.
		#
		#<VirtualHost *:80>
		#    ServerAdmin webmaster@dummy-host.example.com
		#    DocumentRoot /www/docs/dummy-host.example.com
		#    ServerName dummy-host.example.com
		#    ErrorLog logs/dummy-host.example.com-error_log
		#    CustomLog logs/dummy-host.example.com-access_log common
		#</VirtualHost>

- 仿照例子, 添加一段代码来指定某一域名的网站
 
		# DocumentRoot 是网站文件存放的根目录
		# ServerName 是网站域名, 需要跟 DNS 指向的域名一致
		#
		<VirtualHost *:80>
			ServerAdmin webmaster@dummy-host.example.com
			DocumentRoot /var/www/httpdocs/demo_neoease_com
			ServerName demo.neoease.com
			ErrorLog logs/demo.neoease.com-error.log
			CustomLog logs/demo.neoease.com-access.log common
		</VirtualHost>

- 重启 httpd 服务, 执行以下语句

		service httpd restart

##Windows

- 打开目录 {Apache2 安装目录}\conf\extra\, 找到 httpd-vhosts.conf 文件

- 仿照例子, 添加一段代码来指定某一域名的网站

		# DocumentRoot 是网站文件存放的根目录
		# ServerName 是网站域名, 需要跟 DNS 指向的域名一致
		#
		<VirtualHost *:80>
			ServerAdmin webmaster@dummy-host.example.com
			DocumentRoot "D:/workspace/php/demo_neoease_com"
			ServerName demo.neoease.com
			ErrorLog "logs/demo.neoease.com-error.log"
			CustomLog "logs/demo.neoease.com-access.log" common
		</VirtualHost>

- 打开 httpd.conf 文件, 添加如下语句

		# Virtual hosts
		Include conf/extra/httpd-vhosts.conf

- 重启 Apache 服务

##Mac OS

- 打开文件 /private/etc/apache2/extra/httpd-vhosts.conf

- 仿照例子, 添加一段代码来指定某一域名的网站

		# DocumentRoot 是网站文件存放的根目录
		# ServerName 是网站域名, 需要跟 DNS 指向的域名一致
		#
		<VirtualHost *:80>
			ServerAdmin webmaster@dummy-host.example.com
			DocumentRoot "/usr/docs/httpdocs/demo_neoease_com"
			ServerName demo.neoease.com
			ErrorLog "/private/var/log/apache2/demo.neoease.com-error_log"
			CustomLog "/private/var/log/apache2/demo.neoease.com-access_log" common
		</VirtualHost>

- 打开文件 /private/etc/apache2/httpd.conf, 搜索 Virtual hosts, 找到代码如下:

		# Virtual hosts
		#Include /private/etc/apache2/extra/httpd-vhosts.conf

去掉前面的注释符号 #, 保存文件

- 重启 apache 服务, 执行以下语句

		sudo apachectl restart

##Nginx 虚拟主机

- 进入 /usr/local/nginx/conf/vhost 目录, 创建虚拟主机配置文件 demo.neoease.com.conf ({域名}.conf)

- 打开配置文件, 添加服务如下:

		server {
			listen       80;
			server_name demo.neoease.com;
			index index.html index.htm index.php;
			root  /var/www/demo_neoease_com;
		
			log_format demo.neoease.com '$remote_addr - $remote_user [$time_local] $request'
			'$status $body_bytes_sent $http_referer '
			'$http_user_agent $http_x_forwarded_for';
			access_log  /var/log/demo.neoease.com.log demo.neoease.com;
		}

- 打开 Nginx 配置文件 /usr/local/nginx/conf/nginx.conf, 在 http 范围引入虚拟主机配置文件如下:

		include vhost/*.conf;

- 重启 Nginx 服务, 执行以下语句

		service nginx restart

##让 Nginx 虚拟主机支持 PHP

- 在前面第 2 步的虚拟主机服务对应的目录加入对 PHP 的支持, 这里使用的是 FastCGI, 修改如下

		server {
			listen       80;
			server_name demo.neoease.com;
			index index.html index.htm index.php;
			root  /var/www/demo_neoease_com;
		
			location ~ .*\.(php|php5)?$ {
				fastcgi_pass unix:/tmp/php-cgi.sock;
				fastcgi_index index.php;
				include fcgi.conf;
			}
		
			log_format demo.neoease.com '$remote_addr - $remote_user [$time_local] $request'
			'$status $body_bytes_sent $http_referer '
			'$http_user_agent $http_x_forwarded_for';
			access_log  /var/log/demo.neoease.com.log demo.neoease.com;
		}

##图片防盗链

图片作为重要的耗流量大的静态资源, 可能网站主并不希望其他网站直接引用, Nginx 可以通过 referer 来防止外站盗链图片

		server {
			listen       80;
			server_name demo.neoease.com;
			index index.html index.htm index.php;
			root  /var/www/demo_neoease_com;
		
			# 这里为图片添加为期 1 年的过期时间, 并且禁止 Google, 百度和本站之外的网站引用图片
			location ~ .*\.(ico|jpg|jpeg|png|gif)$ {
				expires 1y;
				valid_referers none blocked demo.neoease.com *.google.com *.baidu.com;
				if ($invalid_referer) {
					return 404;
				}
			}
		
			log_format demo.neoease.com '$remote_addr - $remote_user [$time_local] $request'
			'$status $body_bytes_sent $http_referer '
			'$http_user_agent $http_x_forwarded_for';
			access_log  /var/log/demo.neoease.com.log demo.neoease.com;
		}

