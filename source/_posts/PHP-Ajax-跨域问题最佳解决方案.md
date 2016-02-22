title: PHP Ajax 跨域问题最佳解决方案
date: 2015-12-28 16:47:26
tags: [PHP,Ajax]
---
### 本文通过设置Access-Control-Allow-Origin来实现跨域。

例如：客户端的域名是client.runoob.com，而请求的域名是server.runoob.com。

<!-- more -->
如果直接使用ajax访问，会有以下错误：

	XMLHttpRequest cannot load http://server.runoob.com/server.php. No 'Access-Control-Allow-Origin' header is present on the requested resource.Origin 'http://client.runoob.com' is therefore not allowed access.
	
### 1、允许单个域名访问
指定某域名（http://client.runoob.com）跨域访问，则只需在http://server.runoob.com/server.php文件头部添加如下代码：

	header('Access-Control-Allow-Origin:http://client.runoob.com');
### 2、允许多个域名访问
指定多个域名（http://client1.runoob.com、http://client2.runoob.com等）跨域访问，则只需在http://server.runoob.com/server.php文件头部添加如下代码：

```
$origin = isset($_SERVER['HTTP_ORIGIN'])? $_SERVER['HTTP_ORIGIN'] : '';  
  
$allow_origin = array(  
    'http://client1.runoob.com',  
    'http://client2.runoob.com'  
);  
  
if(in_array($origin, $allow_origin)){  
    header('Access-Control-Allow-Origin:'.$origin);       
}
```

### 3、允许所有域名访问
允许所有域名访问则只需在http://server.runoob.com/server.php文件头部添加如下代码：

	header('Access-Control-Allow-Origin:*'); 