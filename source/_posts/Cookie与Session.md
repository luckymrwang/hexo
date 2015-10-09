title: "Cookie与Session"
date: 2015-05-23 10:51:39
tags: [Cookie,Session]
---
## Cookie的基本概念和设置

Cookie在远程浏览器端存储数据并以此跟踪和识别用户的机制。从实现上说，Cookie是存储在客户端上的一小段数据（4k），浏览器（客户端）通过HTTP协议和服务器端进行Cookie交互。

*注意：这里说的是客户端而不是浏览器，实际上能管理Cookie的不仅仅是浏览器，当然最常见的是由浏览器管理Cookie，后面的叙述中不再区分这两个概念。*
<!-- more -->
Cookie独立于语言存在，也就是说，不论是PHP还是JSP种下的Cookie，其本质都是一样的，客户端脚本（如JavaScript）均能读取到。Cookie在很多语言里都有实现，比如PHP、ASP、Java、.Net。严格地说，Cookie并不是由这些语言实现的，而这些语言则是实现对Cookie的间接操作，即发送HTTP指令，浏览器接收到指令便操作Cookie并返回给服务器。因此，Cookie是由浏览器实现和管理的。

在PHP中可以使用setcookie()或setrawcookie()函数设置Cookie。

设置Cookie时需要注意以下几点：
- 这两个函数有一个返回值，如果是FALSE，代表设置失败；如果为TRUE，代表设置成功。但是这个返回值仅供参考，不代表客户端一定能接收到。
- 由PHP在当前页设置的Cookie不能立即生效，要等到下一个页面才能看到。这是由于设置的这个页面里的Cookie由服务器传给客户浏览器，在下一个页面浏览器才能把Cookie从客户的机器里取出传回服务器。如果是JavaScript设置的，是立即生效的。
- Cookie没有显示的删除函数。如果想删除Cookie，应该将Cookie的expire设置为过期时间，如1小时前、1970年等，这回自动护法浏览器的删除机制。

*Cookie是HTTP头的一部分，即先发送或请求Cookie，然后才是data域因此setcookie()等函数必须在其输出数据之前调用，这和header()函数是相同的。*

*[PHP和JavaScript对Cookie的操作]、[Cookie存储机制及应用]、[Cookie跨域与P3P协议]、[本地存储localStorage]具体内容可参照《PHP核心技术与最佳实践》*

## Session的基本概念和设置

和Cookie一样，Session也是一个通用标准，但在不同的语言中实现有所不同。针对Web网站来说，Session指用户在浏览某个网站时，从进入网站到浏览器关闭这段时间内的会话。使用Session可以在网站的上下文不同页面间传递变量、用户身份认证、程序状态记录等。常见的形式就是配合Cookie使用，实现保存用户登录状态功能。和Cookie一样，session_start()必须在程序最开始执行，前面不能有任何输出内容，否则就会出现警告。

*原因：在http传输文本中，规定必须header和content顺序必须是：header在前content在后，并且header的格式必须满足“keyword: value\n”这种格式。
1、在header输出之前有输出内容的话，就会造成对header的错误理解（尽管现在已经能容错了），例如不是满足“keyword: value\n”的格式还好，直接错误了，但是满足“keyword: value\n”这个格式以后，客户端是否按照错误理解，还是按照正确理解？
2、session开启是会隐含的触发是否用header(“Set-Cookie: sid=xxxxxx”),也就是其实还是一个隐式的header调用*

我们看到，HTTP协议本身并不能支持服务器端保存客户端的状态信息。为了解决这一问题，于是引入了Session的概念，用其来保存客户端的状态信息。Session通过一个称为PHPSESSID的Cookie和服务器联系。Session是通过sessionID判断客户端用户的，即Session文件的文件名。

sessionID实际上是在客户端和服务器端之间通过HTTP Request和HTTP Response传来传去。sessionID按照一定的算法生成，必须包含在HTTP Request里面，保证唯一性和随机性，以确保Session的安全。如果没有设置Session的生存周期，sessionID存储在内存中，关闭浏览器后该ID自动注销；重新请求该页面，会重新注册一个sessionID。如果客户端没有禁止Cookie，Cookie在启动Session会话的时候扮演的是存储sessionID和Session生存期的角色。可以手动设置Session的生存期。

对于访问量大的站点，用默认的Session存储方式并不合适，较优的方法是数据库。

## 两者用例
具体实例对比可以[查看这里](http://www.php1.cn/article/9404.html)