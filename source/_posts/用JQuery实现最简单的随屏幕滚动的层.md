title: 用JQuery实现最简单的随屏幕滚动的层
date: 2015-08-16 14:50:23
tags: [JQuery]
---

首先需要一个绝对定位的元素

	<div id="test" style="position:absolute;">test</div>
	
实现方法：随页面滚动（Scroll）事件动态设置元素的CSS，TOP值
<!-- more -->

定位在页面顶端：

	$(window).scroll(function(){
		$('#test').css('top', $(document).scrollTop());
	});
	
定位在页面底端：

	$(window).scroll(function(){
		$('#test').css('top', $(document).scrollTop() + $(window).height() - $('#test').height());
	});
