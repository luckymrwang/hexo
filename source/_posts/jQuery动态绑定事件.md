title: jQuery动态绑定事件
date: 2015-12-07 15:37:45
tags: [jQuery]
---

在jQuery中给动态元素和属性绑定事件的是 `live` 和 `on`，其中 `live` 在jQuery 1.7之后就不推荐使用了。现在主要用 `on` ，使用是要注意， `on` 前面的元素也必须在页面加载的时候就存在于 `DOM` 里面。 动态元素或者元素必须放在 `on` 的第二个参数里面。

<!-- more -->
如下正确用法：

```js
$('#a').on('click', '.b', function(){
```
下面方式对动态元素不起作用，*两种方法等价*：

```js
$('#a').on('click', function(){
$('#a').click(function(){
```