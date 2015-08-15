title: "Apache AllowOverride None 及 Option 详解"
date: 2015-06-03 16:04:03
tags: [Apache]
---

##AllowOverride

AllowOverride参数就是指明Apache服务器是否去找.htaccess文件作为配置文件，如果设置为None,那么服务器将忽略.htacess文件，如果设置为All,那么所有在.htaccess文件里有的指令都将被重写。对于AllowOverride，还可以对它指定如下一些能被重写的指令类型。
<!-- more -->
*通常利用Apache的rewrite模块对 URL 进行重写的时候， rewrite规则会写在 .htaccess 文件里。但要使 apache 能够正常的读取.htaccess 文件的内容，就必须对.htaccess 所在目录进行配置。从安全性考虑，根目录的AllowOverride属性一般都配置成不允许任何Override ，即*

	<Directory /> 
		AllowOverride None 
	</Directory> 

##Options Indexes FollowSymLinks

*Options Indexes FollowSymLinks* 是来控制 Apache 是否显示目录列表

- 默认情况下

		http://localhost:8080/

如果文件根目录里有 index.html，浏览器就会显示 index.html的内容；

如果没有 index.html，浏览器就会显示文件根目录的目录列表，目录列表包括文件根目录下的文件和子目录。

- 禁用方法

1.只需将 Option 中的 Indexes 去掉即可

	<Directory "D:/Apa/blabla">
		Options Indexes FollowSymLinks #---------->Options FollowSymLinks
		AllowOverride None
		Order allow,deny
		Allow from all
	</Directory>
*只需要将上面代码中的 Indexes 去掉，就可以禁止 Apache 显示该目录结构。用户就不会看到该目录下的文件和子目录列表了。
Indexes 的作用就是当该目录下没有 index.html 文件时，就显示目录结构，去掉 Indexes，Apache 就不会显示该目录的列表了。*

2.编辑httpd.conf文件

在Options Indexes FollowSymLinks在Indexes前面加上 – 符号。
即： Options -Indexes FollowSymLinks

*备注：在Indexes前，加 + 代表允许目录浏览；加 – 代表禁止目录浏览*

这样的话就属于整个Apache禁止目录浏览

如果是在虚拟主机中，只要增加如下信息就行：

	<Directory “D:test”>
		Options -Indexes FollowSymLinks
		AllowOverride None
		Order deny,allow
		Allow from all
	</Directory>

3.在根目录的 .htaccess 文件中输入：

	<Files *>
		Options -Indexes
	</Files>

即阻止Apache 将目录结构列表出来。