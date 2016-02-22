title: AngularJS学习笔记（环境配置）
date: 2015-12-28 15:59:26
tags: [AngularJS]
---

### 什么是 AngularJS?

AngularJS 是一个为动态WEB应用设计的结构框架。它能让你使用HTML作为模板语言，通过扩展HTML的语法，让你能更清楚、简洁地构建你的应用组件。它的创新点在于，利用 数据绑定 和 依赖注入，它使你不用再写大量的代码了。这些全都是通过浏览器端的Javascript实现，这也使得它能够完美地和任何服务器端技术结合。
<!-- more -->
### 学习前环境的准备
1、Node.js下载安装
下载地址：http://nodejs.org/
安装完后，一般会把nodeJs的安装路径写入Path环境变量里。(如没有的话，请手动配置)
检查安装是否成功（dos命令窗口）：node --version 或 node -v
安装Testacular单元测试程序: npm install karma

2、git下载安装
下载地址：http://git-scm.com/
安装完后，一般会把nodeJs的安装路径写入Path环境变量里。(如没有的话，请手动配置)
以下命令从Github复制本教程项目的源代码文件（dos命令窗口）：

	git clone git://github.com/angular/angular-phonecat.git
这个命令会在您当前文件夹中建立新文件夹angular-phonecat。

3、服务器的启动命令

	npm start

4、测试服务器的启动命令

	npm test

5、入门学习的例子的获取

```
git checkout -f step-0
git checkout -f step-1
git checkout -f step-2
---
git checkout -f step-11
```

6、例子的运行
cd到angular-phonecat文件夹
启动：
	
	npm start
	
导出例子程序：
	
	git checkout -f step-0
例子代码生产在文件夹angular-phonecat/app下
浏览器中运行例子：
	
	http://localhost:8000/app/index.html