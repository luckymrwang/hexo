title: set_error_hander()重要用法
date: 2015-10-14 19:45:12
tags: [PHP]
toc: true
---
#### set\_error\_handler这个函数的作用是为了防止错误路径泄露

何为错误路径泄露？

我们写程序，难免会有问题，而PHP遇到错误时，就会给出出错脚本的位置、行数和原因

有很多人说，这并没有什么大不了。确实，在调试程序阶段，这确实是没啥的，而且我认为给出错误路径是必要的。

但泄露了实际路径的后果是不堪设想的，对于某些入侵者，这个信息可是非常重要，而事实上现在有很多的服务器都存在这个问题。
 
有些网管干脆把PHP配置文件中的display_errors设置为Off来解决（貌似我们就是这样做的），但本人认为这个方法过于消极。

<!-- more -->
有些时候，我们的确需要PHP返回错误的信息以便调试。而且在出错时也可能需要给用户一个交待，甚至导航到另一页面。
 
那么，有什么解决办法？

PHP从4.1.0开始提供了自定义错误处理句柄的功能函数set_error_handler()，但很少数脚本编写者知道。

#### set\_error\_handler的使用方法如下：

	string set_error_handler ( callback error_handler [, int error_types]) 
	
~~~php
//admin为管理员的身份判定，true为管理员。
//自定义的错误处理函数一定要有这４个输入变量$errno,$errstr,$errfile,$errline，否则无效。
function my_error_handler($errno,$errstr,$errfile,$errline)
{
    //如果不是管理员就过滤实际路径
    if(!admin)
    {
        $errfile=str_replace(getcwd(),"",$errfile);
        $errstr=str_replace(getcwd(),"",$errstr);
    }
    switch($errno)
    {
        case E_ERROR:
        echo "ERROR: [ID $errno] $errstr (Line: $errline of $errfile) \n";
        echo "程序已经停止运行，请联系管理员。";
        //遇到Error级错误时退出脚本
        exit;
        break;

        case E_WARNING:
        echo "WARNING: [ID $errno] $errstr (Line: $errline of $errfile) \n";
        break;

        default:
        //不显示Notice级的错误
        break;
    }
}
~~~

这样就自定义了一个错误处理函数，那么怎么把错误的处理交给这个自定义函数呢？

~~~php
// 应用到类
set_error_handler(array(&$this,"appError"));

//示例的做法
set_error_handler("my_error_handler");
~~~

需要注意的地方：

- E_ERROR、E_PARSE、E_CORE_ERROR、E_CORE_WARNING、 E_COMPILE_ERROR、E_COMPILE_WARNING是不会被这个句柄处理的，也就是会用最原始的方式显示出来。不过出现这些错误都是编 译或PHP内核出错，在通常情况下不会发生。

- 使用set_error_handler()后，error_reporting ()将会失效。也就是所有的错误（除上述的错误）都会交给自定义的函数处理。

#### 示例

~~~php
//先定义一个函数，也可以定义在其他的文件中，再用require()调用
function myErrorHandler($errno, $errstr, $errfile, $errline)
{
　　　　　//为了安全起见，不暴露出真实物理路径，下面两行过滤实际路径
    $errfile=str_replace(getcwd(),"",$errfile);
    $errstr=str_replace(getcwd(),"",$errstr);

    switch ($errno) {
    case E_USER_ERROR:

     echo "<b>My ERROR</b> [$errno] $errstr<br />\n";
        echo "  Fatal error on line $errline in file $errfile";
        echo ", PHP " . PHP_VERSION . " (" . PHP_OS . ")<br />\n";
        echo "Aborting...<br />\n";
        exit(1);
        break;

    case E_USER_WARNING:
        echo "<b>My WARNING</b> [$errno] $errstr<br />\n";
        break;

    case E_USER_NOTICE:
        echo "<b>My NOTICE</b> [$errno] $errstr<br />\n";
        break;

    default:
        echo "Unknown error type: [$errno] $errstr<br />\n";
        break;
    }

    /* Don't execute PHP internal error handler */
    return true;
}

//下面开始连接MYSQL服务器，我们故意指定MYSQL端口为3333,实际为3306。
$link_id=@mysql_pconnect("localhost:3333","root","password");
set_error_handler(myErrorHandler);
if (!$link_id) {
    trigger_error("出错了", E_USER_ERROR);
} 
~~~

#### 总结

~~~php
class CallbackClass {
   function CallbackFunction() {
       // refers to $this
   }

   function StaticFunction() {
       // doesn't refer to $this
   }
}

function NonClassFunction($errno, $errstr, $errfile, $errline) {
}

// 三种方法如下:

1: set_error_handler('NonClassFunction');  // 直接转到一个普通的函数 NonClassFunction

2: set_error_handler(array('CallbackClass', 'StaticFunction')); // 转到 CallbackClass 类下的静方法 StaticFunction

3: $o =& new CallbackClass();
    set_error_handler(array($o, 'CallbackFunction'));  // 转到类的构造函数，其实本质上跟下面的第四条一样。

4. $o = new CallbackClass();


// The following may also prove useful:

class CallbackClass {
   function CallbackClass() {
       set_error_handler(array(&$this, 'CallbackFunction')); // the & is important
   }
   
   function CallbackFunction() {
       // refers to $this
   }
}
~~~