title: FixedHeader随表及浏览器大小自适应
date: 2015-08-13 20:09:42
tags: [DataTables]
---

DataTables中有一个浮动表头的扩展[FixedHeader](http://datatables.net/extensions/fixedheader/),他的作用就是在浏览器滚动时使其表头能够浮动。

使用方法：

在引入js文件后，初始化一下命令：

```js
	$(document).ready( function () {
		var table = $('#example').DataTable();
		new $.fn.dataTable.FixedHeader( table );
	} );
```

<!-- more -->
*它的实现原理就是在表格形成后，复制一份表头覆盖在原来的表头上*	
	
这样初始化后是没有问题的，但是在改变浏览器大小之后，它就一直是默认的长度，不能自适应了。结局办法就是添加如下代码：

```js
	var oFH = new $.fn.dataTable.FixedHeader( table );	$(window).resize(function(){		oFH._fnUpdateClones(true);		oFH._fnUpdatePositions();	});
```*通过JS来监听浏览器窗口改变之后，表的结构已经自适应了，然后它再依据表头重新绘制一份*	
下面这段代码是BI系统中缩小sidebar后使其自适应
```js	$('#sidebar-collapse').click(function(){		oFH._fnUpdateClones(true);		oFH._fnUpdatePositions();	});
```解决办法参考[这里](http://datatables.net/forums/discussion/19124/simple-solution-for-fixedheader-column-headers-not-changing-on-window-resize#)
```js	var fixedHeaders = [];
	$('.table-fixed-header').each(function(){
		fixedHeaders.push(
			new FixedHeader(this, {
			'offsetTop': 51 // offset for my bootstrap .navbar-fixed-top
        	})
    	);
	});

	$(window).resize(function(){
		for(var i = 0; i < fixedHeaders.length; i++){
			fixedHeaders[i]._fnUpdateClones(true); // force redraw
			fixedHeaders[i]._fnUpdatePositions();
		}
	});
```