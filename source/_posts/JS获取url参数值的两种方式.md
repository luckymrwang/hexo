title: JS获取url参数值的两种方式
date: 2015-12-24 20:31:40
tags: [JS]
---

JS获取url：
	window.location.href;
	
有个url格式如下：
http://localhost/lab/GM-center/src/user_manage/user_info?game=tank&appid=1009

### 通过正则
```js
function getQueryString(name) {
	var reg = new RegExp("(^|&)" + name + "=([^&]*)(&|$)", "i");
	var r = window.location.search.substr(1).match(reg);
	if (r != null) return unescape(r[2]); return null;
}

var from = getQueryString("game");

alert(from);
```

### 通过切串放进数组
```js
function GetRequest() { 
	var url = location.search; //获取url中"?"符后的字串 
	var theRequest = new Object(); 
	if (url.indexOf("?") != -1) {
		var str = url.substr(1); 
		strs = str.split("&"); 
		for(var i = 0; i < strs.length; i ++) {
			theRequest[strs[i].split("=")[0]]=unescape(strs[i].split("=")[1]); 
		} 
	} 
	return theRequest; 
} 

var req = GetRequest(); 

var from = req['game'];

alert(from);
```