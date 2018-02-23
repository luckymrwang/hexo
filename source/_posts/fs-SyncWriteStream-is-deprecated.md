title: fs.SyncWriteStream is deprecated
date: 2017-12-28 14:43:25
tags: [Hexo]
toc: true
---
今天在运行`hexo`时又报错了，错误具体信息如下

```
(node:96650) [DEP0061] DeprecationWarning: fs.SyncWriteStream is deprecated.
```
<!-- more -->

难道是我很久不用它来写博客，闹情绪了？

在网上查阅后给出的结果是，`nodejs`从8.0开始已经弃用了`fs.SyncWriteStream`方法，但是某些插件里面还是用到这个方法。查看Hexo项目也有这个一条`issue`，在`hexo`项目中其中有一个`hexo-fs`的插件调用了这个方法，所以需要更新`hexo-fs`插件，更新方法如下：

```
npm install hexo-fs --save
```

更新后于是再次运行`hexo`命令，结果还是报错（尴尬脸）。问题没有解决，`hexo`命令有个`-debug`参数，运行命令的时候加上这个参数，可以定位问题：

```
SinodeMacBook-Pro:hexo sino$ hexo clean --debug
06:39:31.262 DEBUG Hexo version: 3.3.9
06:39:31.264 DEBUG Working directory: ~/Documents/luckymrwang/hexo/
06:39:31.494 DEBUG Config loaded: ~/Documents/luckymrwang/hexo/_config.yml
06:39:31.533 DEBUG Plugin loaded: hexo-deployer-git
(node:96650) [DEP0061] DeprecationWarning: fs.SyncWriteStream is deprecated.
06:39:31.566 DEBUG Plugin loaded: hexo-fs
06:39:31.574 DEBUG Plugin loaded: hexo-generator-archive
06:39:31.576 DEBUG Plugin loaded: hexo-generator-category
06:39:31.578 DEBUG Plugin loaded: hexo-generator-index
06:39:31.581 DEBUG Plugin loaded: hexo-generator-tag
06:39:31.593 DEBUG Plugin loaded: hexo-renderer-ejs
06:39:31.659 DEBUG Plugin loaded: hexo-renderer-marked
06:39:31.661 DEBUG Plugin loaded: hexo-renderer-stylus
06:39:31.851 DEBUG Plugin loaded: hexo-server
06:39:31.861 DEBUG Script loaded: themes/jacman/scripts/fancybox.js
06:39:31.863 INFO  Deleted database.
06:39:31.876 DEBUG Database saved
```

所以问题很明显了，`hexo-deployer-git`这个插件也需要更新，于是按照之前的方式：

```
SinodeMacBook-Pro:hexo sino$ npm install hexo-deployer-git --save
+ hexo-deployer-git@0.1.0
updated 1 package in 13.266s
```
开开心心地再次运行`hexo`，最终依旧报错，此时内心早已崩溃~

于是继续调查问题，看到网上信息一些人的`hexo-deployer-git`版本是`0.3.1`，而我的却是`0.1.0`，于是就尝试升级下插件版本

```
SinodeMacBook-Pro:hexo sino$ npm install hexo-deployer-git@0.3.1 --save
+ hexo-deployer-git@0.3.1
added 17 packages, removed 7 packages and updated 4 packages in 20.742s

```

问题顺利解决，相同问题也会迎刃而解，依此记录。