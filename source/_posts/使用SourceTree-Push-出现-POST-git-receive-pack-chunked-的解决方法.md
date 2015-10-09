title: 使用SourceTree Push 出现 POST git-receive-pack (chunked) 的解决方法
date: 2015-10-07 12:44:00
tags: [GitHub]
---
### 问题描述
在使用SourceTree上传资料的时候,遇到

	POST git-receive-pack (chunked) 
	
<!-- more -->
### 问题调查
从 stackoverflow 看到这样一则

	This is a bug in Git; when using HTTPS it will use chunked encoding for uploads above a certain size. Those do not work. 

	A trivial fix is to tell git to not chunk until some ridiculously large size value, such as: 
	git config http.postBuffer 524288000 
	
### 问题解决办法	
在 git/config 加入

	[http] 
	    postBuffer = 524288000