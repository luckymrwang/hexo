title: HTML DOM position 属性
date: 2016-01-11 13:08:44
tags: [HTML]
toc: true
---
### 定义和用法
`position` 属性把元素放置到一个静态的、相对的、绝对的、或固定的位置中。

### 用法
	Object.style.position=static|relative|absolute|fixed
	
### 可能的值

值  | 描述
------------- | -------------
static  | 默认。位置设置为 static 的元素，它始终会处于页面流给予的位置（static 元素会忽略任何 top、bottom、left 或 right 声明）。
relative  | 位置被设置为 relative 的元素，可将其移至相对于其正常位置的地方，因此 "left:20" 会将元素移至元素正常位置左边 20 个像素的位置。
absolute  | 位置设置为 absolute 的元素，可定位于相对于包含它的元素的指定坐标。此元素的位置可通过 "left"、"top"、"right" 以及 "bottom" 属性来规定。
fixed  | 位置被设置为 fixed 的元素，可定位于相对于浏览器窗口的指定坐标。此元素的位置可通过 "left"、"top"、"right" 以及"bottom" 属性来规定。不论窗口滚动与否，元素都会留在那个位置。工作于 IE7（strict 模式）。

具体信息可以查看[这里](http://zh.learnlayout.com/position.html)