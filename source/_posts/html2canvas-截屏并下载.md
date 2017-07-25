title: html2canvas 截屏并下载
date: 2017-07-24 14:29:21
tags: [html2canvas]
---

[html2canvas](http://html2canvas.hertzen.com/) 可以截取当前屏幕网页的内容并生成图片，官网提供的 [example](http://html2canvas.hertzen.com/examples.html) 只能截图并显示在网页底部，[document](http://html2canvas.hertzen.com/documentation.html)也只是一些参数的使用，下面我利用 `html5` 的新特性将生成的图片自动下载到本地，方法参见 [如何用 JavaScript 下载文件](/2017/07/24/如何用-JavaScript-下载文件/)

引入 `html2canvas.js`

```js
$("#snapshot").click(function(){
	html2canvas($("table"), {
		onrendered: function(canvas) {
			var img = document.createElement('a');
			img.href = canvas.toDataURL("image/jpeg").replace("image/jpeg", "image/octet-stream");
			img.download = 'case.jpg';
			img.click();
		},
		background: "#fff"
	});
})
```

参照资料[stackoverflow](https://stackoverflow.com/questions/31656689/how-to-save-img-to-users-local-computer-using-html2canvas)

