title: 如何做到jQuery-free?
date: 2016-02-29 11:29:35
tags: [jQuery]
toc: true
---
jQuery是现在最流行的JavaScript工具库。

据统计，目前全世界57.3%的网站使用它。也就是说，10个网站里面，有6个使用jQuery。如果只考察使用工具库的网站，这个比例就会上升到惊人的91.7%。

虽然jQuery如此受欢迎，但是它臃肿的体积也让人头痛不已。jQuery 2.0的原始大小为235KB，优化后为81KB；如果是支持IE6、7、8的jQuery 1.8.3，原始大小为261KB，优化后为91KB。

<!-- more -->
这样的体积，即使是宽带环境，完全加载也需要1秒或更长，更不要说移动设备了。这意味着，如果你使用了jQuery，用户至少延迟1秒，才能看到网页效果。考虑到本质上，jQuery只是一个操作DOM的工具，我们不仅要问：如果只是为了几个网页特效，是否有必要动用这么大的库？

2006年，jQuery诞生的时候，主要用于消除不同浏览器的差异（主要是IE6），为开发者提供一个简洁的统一接口。相比当时，如今的情况已经发生了很大的变化。IE的市场份额不断下降，以ECMAScript为基础的JavaScript标准语法，正得到越来越广泛的支持。开发者直接使用JavScript标准语法，就能同时在各大浏览器运行，不再需要通过jQuery获取兼容性。

下面就探讨如何用JavaScript标准语法，取代jQuery的一些主要功能，做到jQuery-free。

### 选取DOM元素
jQuery的核心是通过各种选择器，选中DOM元素，可以用querySelectorAll方法模拟这个功能。

```js
var $ = document.querySelectorAll.bind(document);
```

这里需要注意的是，querySelectorAll方法返回的是NodeList对象，它很像数组（有数字索引和length属性），但不是数组，不能使用pop、push等数组特有方法。如果有需要，可以考虑将Nodelist对象转为数组。

```js
myList = Array.prototype.slice.call(myNodeList);
```

### DOM操作
DOM本身就具有很丰富的操作方法，可以取代jQuery提供的操作方法。

尾部追加DOM元素。

```js
　　// jQuery写法
　　$(parent).append($(child));
　　// DOM写法
　　parent.appendChild(child)
```

头部插入DOM元素。

```js
	// jQuery写法
　　$(parent).prepend($(child));
　　// DOM写法
　　parent.insertBefore(child, parent.childNodes[0])
```

删除DOM元素。

```js
　　// jQuery写法
　　$(child).remove()
　　// DOM写法
　　child.parentNode.removeChild(child)
```

### 事件的监听
jQuery的on方法，完全可以用addEventListener模拟。

```
Element.prototype.on = Element.prototype.addEventListener;
```

为了使用方便，可以在NodeList对象上也部署这个方法。

```js
　　NodeList.prototype.on = function (event, fn) {
　　　　[]['forEach'].call(this, function (el) {
　　　　　　el.on(event, fn);
　　　　});
　　　　return this;
　　};
```

### 事件的触发

jQuery的trigger方法则需要单独部署，相对复杂一些。

```js
　　Element.prototype.trigger = function (type, data) {
　　　　var event = document.createEvent('HTMLEvents');
　　　　event.initEvent(type, true, true);
　　　　event.data = data || {};
　　　　event.eventName = type;
　　　　event.target = this;
　　　　this.dispatchEvent(event);
　　　　return this;
　　};
```

在NodeList对象上也部署这个方法。

```js
　　NodeList.prototype.trigger = function (event) {
　　　　[]['forEach'].call(this, function (el) {
　　　　　　el['trigger'](event);
　　　　});
　　　　return this;
　　};
```

### document.ready

目前的最佳实践，是将JavaScript脚本文件都放在页面底部加载。这样的话，其实document.ready方法（jQuery简写为
$(function)）已经不必要了，因为等到运行的时候，DOM对象已经生成了。

### attr方法

jQuery使用attr方法，读写网页元素的属性。

```js
$("#picture").attr("src", "http://url/to/image");
```

DOM元素允许直接读取属性值，写法要简洁许多。

```js
$("#picture").src = "http://url/to/image";
```

需要注意，input元素的this.value返回的是输入框中的值，链接元素的this.href返回的是绝对URL。如果需要用到这两个网页元素的属性准确值，可以用this.getAttribute('value')和this.getAttibute('href')。

### addClass方法

jQuery的addClass方法，用于为DOM元素添加一个class。

```js
$('body').addClass('hasJS');
```

DOM元素本身有一个可读写的className属性，可以用来操作class。

```js
　　document.body.className = 'hasJS';
　　// or
　　document.body.className += ' hasJS';
```

HTML 5还提供一个classList对象，功能更强大（IE 9不支持）。

```js
　　document.body.classList.add('hasJS');
　　document.body.classList.remove('hasJS');
　　document.body.classList.toggle('hasJS');
　　document.body.classList.contains('hasJS');
```

### CSS

jQuery的css方法，用来设置网页元素的样式。

```js
$(node).css( "color", "red" );
```

DOM元素有一个style属性，可以直接操作。

```js
　　element.style.color = "red";;
　　// or
　　element.style.cssText += 'color:red';
```

### 数据储存

jQuery对象可以储存数据。

```js
$("body").data("foo", 52);
```

HTML 5有一个dataset对象，也有类似的功能（IE 10不支持），不过只能保存字符串。

```js
　　element.dataset.user = JSON.stringify(user);
　　element.dataset.score = score;
```

### Ajax

jQuery的Ajax方法，用于异步操作。

```js
　　$.ajax({
　　　　type: "POST",
　　　　url: "some.php",
　　　　data: { name: "John", location: "Boston" }
　　}).done(function( msg ) {
　　　　alert( "Data Saved: " + msg );
　　});
```

我们可以定义一个request函数，模拟Ajax方法。

```js
　　function request(type, url, opts, callback) {
　　　　var xhr = new XMLHttpRequest();
　　　　if (typeof opts === 'function') {
　　　　　　callback = opts;
　　　　　　opts = null;
　　　　}
　　　　xhr.open(type, url);
　　　　var fd = new FormData();
　　　　if (type === 'POST' && opts) {
　　　　　　for (var key in opts) {
　　　　　　　　fd.append(key, JSON.stringify(opts[key]));
　　　　　　}
　　　　}
　　　　xhr.onload = function () {
　　　　　　callback(JSON.parse(xhr.response));
　　　　};
　　　　xhr.send(opts ? fd : null);
　　}
```

然后，基于request函数，模拟jQuery的get和post方法。

```js
　　var get = request.bind(this, 'GET');
　　var post = request.bind(this, 'POST');
```

### 动画

jQuery的animate方法，用于生成动画效果。

```js
$foo.animate('slow', { x: '+=10px' })；
```

jQuery的动画效果，很大部分基于DOM。但是目前，CSS 3的动画远比DOM强大，所以可以把动画效果写进CSS，然后通过操作DOM元素的class，来展示动画。

```js
foo.classList.add('animate')；
```

如果需要对动画使用回调函数，CSS 3也定义了相应的事件。

```js
　　el.addEventListener("webkitTransitionEnd", transitionEnded);
　　el.addEventListener("transitionend", transitionEnded);
```

### 替代方案

由于jQuery体积过大，替代方案层出不穷。

其中，最有名的是[zepto.js](http://zeptojs.com/)。它的设计目标是以最小的体积，做到最大兼容jQuery的API。zepto.js 1.0版的原始大小是55KB，优化后是29KB，gzip压缩后为10KB。

如果不求最大兼容，只希望模拟jQuery的基本功能，那么，[min.js](https://github.com/remy/min.js)优化后只有200字节，而[dolla](https://github.com/lelandrichardson/dolla)优化后是1.7KB。

此外，jQuery本身采用模块设计，可以只选择使用自己需要的模块。具体做法参见它的github网站，或者使用专用的Web界面。

本文引自[阮一峰](http://www.ruanyifeng.com/blog/2013/05/jquery-free.html)
