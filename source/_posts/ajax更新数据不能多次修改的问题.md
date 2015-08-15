title: ajax更新数据不能多次修改的问题
date: 2015-07-13 19:27:34
tags: [Jquery]
---

##问题描述

双击Table单元格修改内容，利用ajax局部更新后，再次点击就不能修改了

##解决办法

原来方法

	$(".table_txt").dblclick(function(){
<!-- more -->
修改后的方法

	$(".scrollBody").on("dblclick",".table_txt",function(){

##说明

使用 on() 方法添加的事件处理程序适用于当前及 `未来的元素（比如由ajax创建的新元素）` 。

##语法

	$(selector).on(event,childSelector,data,function,map)

其中

`childSelector`	可选。规定只能添加到指定的子元素上的事件处理程序（且不是选择器本身），所以若将.table_txt作为子元素，就要取其父元素 .scrollBody作为选择器
