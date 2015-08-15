title: 使用DataTables提示warning的解决办法
date: 2015-07-07 10:07:28
tags: [DataTables]
---
在使用DataTables插件时打开一些页面后会提示warning信息，内容如下：

![warning](/images/warning.jpg)

官网给的解释信息，[点击这里](http://datatables.net/manual/tech-notes/4)
<!-- more -->
原因是有些单元格中的数据为空，DataTables默认会显示所有信息，但对于这种空的信息不知如何处理，所以会给予这种提示。

解决办法：

使用`columns.defaultContent`来默认赋值，如下：

	"columnDefs": [
		{"defaultContent": 0, "targets": ['_all']}
	]
	
*targets* 的值 *_all* 是指所有的列。