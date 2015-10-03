title: "Mac 下搭建 Hexo"
date: 2015-06-04 13:14:58
tags: [Mac,Hexo]
---

## 流程

- 首先 Hexo 是基于 Node.js 的，所以必须安装 Node.js

- 安装 Node.js 方法很多，这里选择 Homebrew 安装方式

- Mac 自带 Ruby 脚本功能，执行 Ruby 语句
<!-- more -->

- Hexo 提交部署 Github 需要使用 Git 工具，所以需要安装 Git，用Homebrew 安装

## 具体详情

### 1.安装Homebrew

	ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)”

### 2.安装Node.js

**2.1** 第一种方式，Homebrew安装

	brew install node

**2.2** 第二种方式，前提是已经安装好Xcode和git *安装git方法在下面介绍*

	git clone git://github.com/joyent/node.git
	cd node
	./configure
	make
	sudo make install

**2.3** 第三种方式，下载源码(http://nodejs.org/download/)，编译执行同上

###3.安装Hexo

**3.1** 第一种方式，用node.js自带npm安装

	npm install -g hexo
	hexo init
	npm install

**3.2** 第二种方式，下载源码(http://www.nodejs.org/download/)，编译执行

###4.安装git

**4.1** 第一种方式，Homebrew安装
	
	sudo brew install git

**4.2** 第二种方式，前提是已经安装好Xcode

	curl -O http://kernel.org/pub/software/scm/git/git-1.7.5.tar.bz2
	tar xjvf git-1.7.4.1.tar.bz2
	cd git-1.7.4.1
	./configure --prefix=/usr/local
	make
	sudo make install
	which git

**4.3** 第三种方式，下载源码(https://www.kernel.org/pub/software/scm/git/)，编译执行同上

**4.4** git配置，[参照这里](http://luckymrwang.github.io/2015/05/16/Generating-SSH-keys/)

*设置个人配置：*

	git config --global user.name "treason258”
	git config --global user.email ma.tengfei2008@163.com

Hexo的安装到此结束，配置可以[参照这里](http://luckymrwang.github.io/2015/05/14/%E6%90%AD%E5%BB%BA%E7%8B%AC%E7%AB%8B%E5%8D%9A%E5%AE%A2%E2%80%94%E2%80%94%E7%AE%80%E6%98%8EGithub-Pages%E4%B8%8EHexo%E6%95%99%E7%A8%8B/)