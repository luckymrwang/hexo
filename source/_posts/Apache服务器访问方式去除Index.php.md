title: "Apache服务器访问方式去除Index.php"
date: 2015-06-03 15:49:30
tags: [Apache]
---

*例如: 你原来的路径是： localhost/index.php/index 
改变后的路径是： localhost/index*

- httpd.conf配置文件中加载了mod_rewrite.so模块 //在Apache里面去配置

		#LoadModule rewrite_module modules/mod_rewrite.so  //把前面的#号去掉 
<!-- more -->
- 在Apache里面去配置，将里面的AllowOverride None都改为AllowOverride All
		
	*不建议在 httpd.conf 中修改， hpptd.conf 最好保持如下：*

		<Directory />
		    Options FollowSymLinks
			AllowOverride None
		</Directory>

	*通过 Include ,如： Include conf.d/*.conf 来在自定义的配置文件中修改：*

		<Directory "/data/websites/*">
			Options -Indexes FollowSymLinks
			AllowOverride All
			Order allow,deny
			Allow from all
		</Directory>
		
		VirtualHost *:80>
			ServerName opsnode.raysns.com
			DocumentRoot /data/websites/opsnode.raysns.com/
		/VirtualHost>

**注意：修改之后一定要重启apache服务。 **

- .htaccess文件必须放到跟目录下 

	这个文件里面加： 

		RewriteEngine on 
		RewriteCond %{REQUEST_FILENAME} !-d 
		RewriteCond %{REQUEST_FILENAME} !-f 
		RewriteRule ^(.*)$ index.php/$1 [L]

	*补充: windows 里面不能创建 .htaccess ，可以这样创建：新建任何一个文件，然后打开， 点击另存为 (文件类型选择所有)，这样就可以创建了*
	