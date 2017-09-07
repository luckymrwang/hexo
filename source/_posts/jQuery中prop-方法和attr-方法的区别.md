title: jQuery中prop()方法和attr()方法的区别
date: 2015-12-05 20:58:09
tags: [jQuery]
---

jquery1.6中新加了一个方法prop()，一直没用过它，官方解释只有一句话:获取在匹配的元素集中的第一个元素的属性值。

官方例举的例子感觉和attr()差不多，也不知道有什么区别，既然有了prop()这个新方法，不可能没用吧，那什么时候该用attr()，什么时候该用prop()呢？

大家都知道有的浏览器只要写disabled，checked就可以了，而有的要写成disabled = "disabled"，checked="checked"，比如用attr("checked")获取checkbox的checked属性时选中的时候可以取到值,值为"checked"但没选中获取值就是undefined。

<!-- more -->
jq提供新的方法“prop”来获取这些属性，就是来解决这个问题的，以前我们使用attr获取checked属性时返回"checked"和"",现在使用prop方法获取属性则统一返回true和false。

那么，什么时候使用attr()，什么时候使用prop()？
1.添加属性名称该属性就会生效应该使用prop();
2.是有true,false两个属性使用prop();
3.其他则使用attr();
项目中jquery升级的时候大家要注意这点！

官方的对比解释[点击这里](http://stackoverflow.com/questions/5874652/prop-vs-attr)

以下是官方建议attr(),prop()的使用：
![propattr](/images/propattr.png)

本文引自[WEB前段开发](http://www.candoudou.com/archives/161)