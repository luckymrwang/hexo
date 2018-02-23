title: CGI、FAST-CGI、PHP-FPM’
date: 2017-11-13 11:54:52
tags:
---
讲Fastcgi之前需要先讲CGI，CGI是为了保证web server传递过来的数据是标准格式的，它是一个协议，方便CGI程序的编写者。Fastcgi是CGI的更高级的一种方式，是用来提高CGI程序性能的。

web server（如nginx）只是内容的分发者。比如，如果请求/index.html，那么web server会去文件系统中找到这个文件，发送给浏览器，这里分发的是静态资源。

<!-- more -->
如果现在请求的是/index.php，根据配置文件，nginx知道这个不是静态文件，需要去找PHP解析器来处理，那么他会把这个请求简单处理后交给PHP解析器。此时CGI便是规定了要传什么数据／以什么格式传输给php解析器的协议。

当web server收到/index.php这个请求后，会启动对应的CGI程序，这里就是PHP的解析器。接下来PHP解析器会解析php.ini文件，初始化执行环境，然后处理请求，再以CGI规定的格式返回处理后的结果，退出进程。web server再把结果返回给浏览器。

那么CGI相较于Fastcgi而言其性能瓶颈在哪呢？CGI针对每个http请求都是fork一个新进程来进行处理，处理过程包括解析php.ini文件，初始化执行环境等，然后这个进程会把处理完的数据返回给web服务器，最后web服务器把内容发送给用户，刚才fork的进程也随之退出。 如果下次用户还请求动态资源，那么web服务器又再次fork一个新进程，周而复始的进行。

而Fastcgi则会先fork一个master，解析配置文件，初始化执行环境，然后再fork多个worker。当请求过来时，master会传递给一个worker，然后立即可以接受下一个请求。这样就避免了重复的劳动，效率自然是高。而且当worker不够用时，master可以根据配置预先启动几个worker等着；当然空闲worker太多时，也会停掉一些，这样就提高了性能，也节约了资源。这就是Fastcgi的对进程的管理。大多数Fastcgi实现都会维护一个进程池。注：swoole作为httpserver，实际上也是类似这样的工作方式。

那PHP-FPM又是什么呢？它是一个实现了Fastcgi协议的程序,用来管理Fastcgi起的进程的,即能够调度php-cgi进程的程序。现已在PHP内核中就集成了PHP-FPM，使用--enalbe-fpm这个编译参数即可。另外，修改了php.ini配置文件后，没办法平滑重启，需要重启php-fpm才可。此时新fork的worker会用新的配置，已经存在的worker继续处理完手上的活。

本文引自[这里](http://www.lxlxw.me/?p=216)