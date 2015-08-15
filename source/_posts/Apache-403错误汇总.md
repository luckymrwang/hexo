title: "Apache 403错误汇总"
date: 2015-06-04 12:51:16
tags: [Apache,Mac]
---

环境是在Macbook Pro中自带的Apache,路径为 **/etc/apache2**

##现象

- 在浏览器中显示：

	Forbidden You don't have permission to access / on this server.
<!-- more -->
- 错误日志中，显示：

	access to /bi/ga/applist denied (filesystem path '/Users/sino/Documents/gitlab') because search permissions are missing on a component of the path

##排查

1. **http.conf**配置文件中的**DocumentRoot** 已经设置好

2. **<Directory /..>**中的路径也与DocumentRoot一样，而且内容为

		<Directory />
			Options FollowSymLinks
			AllowOverride All
			Order deny,allow
			Allow from all
		</Directory>

也是没有问题的，所以下一步就要考虑目录权限的问题了
*用户组和用户也已经按apahce的配置文件里定义的一样了*

首先是目录的基本权限，而其中要注意的，**目录树都应该拥有这些权限**，我设置的是：

	chmod 755 -R /Users/sino/Documents/gitlab/*

看似没有问题，但根据上面这条理论，必须保证 **/Users**、**/Users/sino**、**/Users/sino/Documents** 这几个层级的目录都是755权限，然而我的Documents权限却不是这样，所以修改后再次测试时已经正常了。

##总结

1. 查看错误日志
2. linux系统中文件的权限要特别注意
3. 上网搜索时，不要一味地使用一套关键词，而是多多更换可能的关键词，或者更多使用Google