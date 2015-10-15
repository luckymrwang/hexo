title: PHP错误异常处理详解
date: 2015-10-13 17:40:57
tags: [PHP]
toc: true
---

异常处理（又称为错误处理）功能提供了处理程序运行时出现的错误或异常情况的方法。

异常处理通常是防止未知错误产生所采取的处理措施。异常处理的好处是你不用再绞尽脑汁去考虑各种错误，这为处理某一类错误提供了一个很有效的方法，使编程效率大大提高。当异常被触发时，通常会发生：

- 当前代码状态被保存
- 代码执行被切换到预定义的异常处理器函数

<!-- more -->
根据情况，处理器也许会从保存的代码状态重新开始执行代码，终止脚本执行，或从代码中另外的位置继续执行脚本

PHP 5 提供了一种新的面向对象的错误处理方法。可以使用检测（try）、抛出（throw）和捕获（catch）异常。即使用try检测有没有抛出（throw）异常，若有异常抛出（throw），使用catch捕获异常。


一个 try 至少要有一个与之对应的 catch。定义多个 catch 可以捕获不同的对象。PHP 会按这些 catch 被定义的顺序执行，直到完成最后一个为止。而在这些 catch 内，又可以抛出新的异常。

### 异常的使用

当一个异常被抛出时，其后的代码将不会继续执行，PHP 会尝试查找匹配的 "catch" 代码块。如果一个异常没有被捕获，而且又没用使用 [set\_exception\_handler()](http://php.net/manual/zh/function.set-exception-handler.php) 作相应的处理的话，那么 PHP 将会产生一个严重的错误，并且输出未能捕获异常(Uncaught Exception ... )的提示信息。

抛出异常，但不去捕获它：

~~~php
<?php  
	ini_set('display_errors', 'On');  
	error_reporting(E_ALL & ~ E_WARNING);  
	$error = 'Always throw this error';  
	throw new Exception($error);  
	// 继续执行  
	echo 'Hello World';  
?> 
~~~ 
上面的代码会获得类似这样的一个致命错误：

```
Fatal error: Uncaught exception 'Exception' with message 'Always throw this error' in E:\sngrep\index.php on line 5  
Exception: Always throw this error in E:\sngrep\index.php on line 5  
Call Stack:  
    0.0005     330680   1. {main}() E:\sngrep\index.php:0
```

### Try, throw 和 catch

要避免上面这个致命错误，可以使用try catch捕获掉。
处理处理程序应当包括：

Try - 使用异常的函数应该位于 "try" 代码块内。如果没有触发异常，则代码将照常继续执行。但是如果异常被触发，会抛出一个异常。
       Throw - 这里规定如何触发异常。每一个 "throw" 必须对应至少一个 "catch"
       Catch - "catch" 代码块会捕获异常，并创建一个包含异常信息的对象
       抛出异常并捕获掉，可以继续执行后面的代码：

~~~php
<?php  
try {  
    $error = 'Always throw this error';  
    throw new Exception($error);  
  
    // 从这里开始，tra 代码块内的代码将不会被执行  
    echo 'Never executed';  
  
} catch (Exception $e) {  
    echo 'Caught exception: ',  $e->getMessage(),'<br>';  
}  
  
// 继续执行  
echo 'Hello World';  
?>   
~~~

在 "try" 代码块检测有有没有抛出“throw”异常，这里抛出了异常。
    "catch" 代码块接收到该异常，并创建一个包含异常信息的对象 ($e)。
    通过从这个 exception 对象调用 $e->getMessage()，输出来自该异常的错误消息
    为了遵循“每个 throw 必须对应一个 catch”的原则，可以设置一个顶层的异常处理器来处理漏掉的错误。


### 扩展 PHP 内置的异常处理类

用户可以用自定义的异常处理类来扩展 PHP 内置的异常处理类。以下的代码说明了在内置的异常处理类中，哪些属性和方法在子类中是可访问和可继承的。（注：以下这段代码只为说明内置异常处理类的结构，它并不是一段有实际意义的可用代码。）

~~~php
<?php  
class Exception  
{  
    protected $message = 'Unknown exception';   // 异常信息  
    protected $code = 0;                        // 用户自定义异常代码  
    protected $file;                            // 发生异常的文件名  
    protected $line;                            // 发生异常的代码行号  
  
    function __construct($message = null, $code = 0);  
  
    final function getMessage();                // 返回异常信息  
    final function getCode();                   // 返回异常代码  
    final function getFile();                   // 返回发生异常的文件名  
    final function getLine();                   // 返回发生异常的代码行号  
    final function getTrace();                  // backtrace() 数组  
    final function getTraceAsString();          // 已格成化成字符串的 getTrace() 信息  
  
    /* 可重载的方法 */  
    function __toString();                       // 可输出的字符串  
}
~~~

如果使用自定义的类来扩展内置异常处理类，并且要重新定义构造函数的话，建议同时调用 parent::__construct() 来检查所有的变量是否已被赋值。当对象要输出字符串的时候，可以重载__toString() 并自定义输出的样式。 

构建自定义异常处理类：

~~~php
<?php  
  
/** 
 *  
 * 自定义一个异常处理类 
 */  
  
class MyException extends Exception  
{  
    // 重定义构造器使 message 变为必须被指定的属性  
    public function __construct($message, $code = 0) {  
        // 自定义的代码  
  
        // 确保所有变量都被正确赋值  
        parent::__construct($message, $code);  
    }  
  
    // 自定义字符串输出的样式 */  
    public function __toString() {  
        return __CLASS__ . ": [{$this->code}]: {$this->message}\n";  
    }  
  
    public function customFunction() {  
        echo "A Custom function for this type of exception\n";  
    }  
}

// 例子 1:抛出自定义异常,但没有默认的异常  
echo ' 例子 1', '<br>';  
try {  
    // 抛出自定义异常  
    throw new MyException('1 is an invalid parameter', 5);  
} catch (MyException $e) {      // 捕获异常  
    echo "Caught my exception\n", $e;  
    $e->customFunction();  
} catch (Exception $e) {        // 被忽略  
    echo "Caught Default Exception\n", $e;  
}  
// 执行后续代码  
// 例子 2： 抛出默认的异常  但没有自定义异常  
echo '<br>', ' 例子 2:', '<br>';  
try {  
     // 抛出默认的异常    
    throw new Exception('2 isnt allowed as a parameter', 6);  
} catch (MyException $e) {      // 不能匹配异常的种类，被忽略  
    echo "Caught my exception\n", $e;  
    $e->customFunction();  
} catch (Exception $e) {// 捕获异常  
    echo "Caught Default Exception\n", $e;  
}  
// 执行后续代码  
// 例子 3: 抛出自定义异常 ，使用默认异常类对象来捕获  
echo '<br>', ' 例子 3:', '<br>';  
try {  
     // 抛出自定义异常   
    throw new MyException('3 isnt allowed as a parameter', 6);  
} catch (Exception $e) {        // 捕获异常  
    echo "Default Exception caught\n", $e;  
}  
  
// 执行后续代码  
// 例子 4  
echo '<br>', ' 例子 4:', '<br>';  
try {  
    echo 'No Exception ';  
} catch (Exception $e) {        // 没有异常，被忽略  
    echo "Default Exception caught\n", $e;  
}  
  
// 执行后续代码
~~~

MyException 类是作为旧的 exception 类的一个扩展来创建的。这样它就继承了旧类的所有属性和方法，我们可以使用 exception 类的方法，比如 getLine() 、 getFile() 以及 getMessage()。

### 嵌套异常处理

如果在内层 "try" 代码块中异常没有被捕获，则它将在外层级上查找 catch 代码块去捕获。

~~~php
try {  
    try {  
    throw new MyException('foo!');  
    } catch (MyException $e) {  
        /* 重新抛出 rethrow it */  
         $e->customFunction();  
        throw $e;  
        
     }  
} catch (Exception $e) {  
        var_dump($e->getMessage());  
}  
~~~

### 设置顶层异常处理器 （Top Level Exception Handler）

set_exception_handler() 函数可设置处理所有未捕获异常的用户定义函数。  

~~~php
<?php  
function myException($exception)  
{  
echo "<b>Exception:</b> " , $exception->getMessage();  
}  
  
set_exception_handler('myException');  
throw new Exception('Uncaught Exception occurred');  
     输出结果：

Exception: Uncaught Exception occurred  
~~~

### 异常的规则

- 需要进行异常处理的代码应该放入 try 代码块内，以便捕获潜在的异常。
- 每个 try 或 throw 代码块必须至少拥有一个对应的 catch 代码块。
- 使用多个 catch 代码块可以捕获不同种类的异常。
- 可以在 try 代码块内的 catch 代码块中再次抛出（re-thrown）异常。

简而言之：如果抛出了异常，就必须捕获它,否则程序终止执行。

在我们实际开发中，错误及异常捕捉仅仅靠try{}catch()是远远不够的。

set_error_handler
一般用于捕捉  `E_NOTICE` 、`E_USER_ERROR`、`E_USER_WARNING`、`E_USER_NOTICE`

不能捕捉：
`E_ERROR`, `E_PARSE`, `E_CORE_ERROR`, `E_CORE_WARNING`, `E_COMPILE_ERROR` and `E_COMPILE_WARNING`。

一般与trigger_error("...", E_USER_ERROR)，配合使用。



### PHP错误处理

在实际开发中，错误及异常捕捉仅仅靠try{}catch()是远远不够的。
所以引用以下几中函数。

#### set_error_handler

一般用于捕捉  E_NOTICE 、E_USER_ERROR、E_USER_WARNING、E_USER_NOTICE

不能捕捉：
E_ERROR, E_PARSE, E_CORE_ERROR, E_CORE_WARNING, E_COMPILE_ERROR and E_COMPILE_WARNING。

一般与trigger_error("...", E_USER_ERROR)，配合使用。

~~~php
<?php  
// we will do our own error handling  
error_reporting(0);  
function userErrorHandler($errno, $errmsg, $filename, $linenum, $vars)  
{  
    // timestamp for the error entry      
    $dt = date("Y-m-d H:i:s (T)");      
    // define an assoc array of error string      
    // in reality the only entries we should      
    // consider are E_WARNING, E_NOTICE, E_USER_ERROR,      
    // E_USER_WARNING and E_USER_NOTICE      
    $errortype = array (                  
        E_ERROR              => 'Error',                  
        E_WARNING            => 'Warning',                  
        E_PARSE              => 'Parsing Error',                  
        E_NOTICE             => 'Notice',                  
        E_CORE_ERROR         => 'Core Error',                  
        E_CORE_WARNING       => 'Core Warning',                  
        E_COMPILE_ERROR      => 'Compile Error',                  
        E_COMPILE_WARNING    => 'Compile Warning',                  
        E_USER_ERROR         => 'User Error',                  
        E_USER_WARNING       => 'User Warning',                  
        E_USER_NOTICE        => 'User Notice',                  
        E_STRICT             => 'Runtime Notice',                  
        E_RECOVERABLE_ERROR  => 'Catchable Fatal Error'                  
    );      
    // set of errors for which a var trace will be saved      
    $user_errors = array(E_USER_ERROR, E_USER_WARNING, E_USER_NOTICE);          
    $err = "<errorentry>\n";      
    $err .= "\t<datetime>" . $dt . "</datetime>\n";      
    $err .= "\t<errornum>" . $errno . "</errornum>\n";      
    $err .= "\t<errortype>" . $errortype[$errno] . "</errortype>\n";      
    $err .= "\t<errormsg>" . $errmsg . "</errormsg>\n";      
    $err .= "\t<scriptname>" . $filename . "</scriptname>\n";      
    $err .= "\t<scriptlinenum>" . $linenum . "</scriptlinenum>\n";      
    if (in_array($errno, $user_errors)) {          
        $err .= "\t<vartrace>" . wddx_serialize_value($vars, "Variables") . "</vartrace>\n";      
    }      
    $err .= "</errorentry>\n\n";  
    echo $err;  
}  
function distance($vect1, $vect2) {      
    if (!is_array($vect1) || !is_array($vect2)) {          
        trigger_error("Incorrect parameters, arrays expected", E_USER_ERROR);          
        return NULL;      
    }      
    if (count($vect1) != count($vect2)) {          
        trigger_error("Vectors need to be of the same size", E_USER_ERROR);          
        return NULL;      
    }   
    for ($i=0; $i<count($vect1); $i++) {          
        $c1 = $vect1[$i]; $c2 = $vect2[$i];          
        $d = 0.0;          
        if (!is_numeric($c1)) {              
        trigger_error("Coordinate $i in vector 1 is not a number, using zero",E_USER_WARNING);              
        $c1 = 0.0;          
    }          
    if (!is_numeric($c2)) {              
        trigger_error("Coordinate $i in vector 2 is not a number, using zero",E_USER_WARNING);              
        $c2 = 0.0;          
    }  
    $d += $c2*$c2 - $c1*$c1;      
    }      
    return sqrt($d);  
}  
  
$old_error_handle = set_error_handler("userErrorHandler");  
$t = I_AM_NOT_DEFINED;  //generates a warning  
  
// define some "vectors"  
$a = array(2, 3, "foo");  
$b = array(5.5, 4.3, -1.6);  
$c = array(1, -3);  
  
//generate a user error  
$t1 = distance($c,$b);  
  
// generate another user error  
$t2 = distance($b, "i am not an array") . "\n";  
  
// generate a warning  
$t3 = distance($a, $b) . "\n";  
?>
~~~

#### set_exception_handler 

设置默认的异常处理程序，用于没有用 try/catch 块来捕获的异常。 在 exception_handler 调用后异常会中止。 
与throw new Exception('Uncaught Exception occurred')，连用。

~~~php
<?php  
// we will do our own error handling  
error_reporting(0);  
function exceptHandle($errno, $errmsg, $filename, $linenum, $vars)  
{  
    // timestamp for the error entry      
    $dt = date("Y-m-d H:i:s (T)");      
    // define an assoc array of error string      
    // in reality the only entries we should      
    // consider are E_WARNING, E_NOTICE, E_USER_ERROR,      
    // E_USER_WARNING and E_USER_NOTICE      
    $errortype = array (                  
        E_ERROR              => 'Error',                  
        E_WARNING            => 'Warning',                  
        E_PARSE              => 'Parsing Error',                  
        E_NOTICE             => 'Notice',                  
        E_CORE_ERROR         => 'Core Error',                  
        E_CORE_WARNING       => 'Core Warning',                  
        E_COMPILE_ERROR      => 'Compile Error',                  
        E_COMPILE_WARNING    => 'Compile Warning',                  
        E_USER_ERROR         => 'User Error',                  
        E_USER_WARNING       => 'User Warning',                  
        E_USER_NOTICE        => 'User Notice',                  
        E_STRICT             => 'Runtime Notice',                  
        E_RECOVERABLE_ERROR  => 'Catchable Fatal Error'                  
    );      
    // set of errors for which a var trace will be saved      
    $err = "<errorentry>\n";      
    $err .= "\t<datetime>" . $dt . "</datetime>\n";      
    $err .= "\t<errornum>" . $errno . "</errornum>\n";      
    $err .= "\t<errortype>" . $errortype[$errno] . "</errortype>\n";      
    $err .= "\t<errormsg>" . $errmsg . "</errormsg>\n";      
    $err .= "\t<scriptname>" . $filename . "</scriptname>\n";      
    $err .= "\t<scriptlinenum>" . $linenum . "</scriptlinenum>\n";      
    if (1) {          
        $err .= "\t<vartrace>" . wddx_serialize_value($vars, "Variables") . "</vartrace>\n";      
    }      
    $err .= "</errorentry>\n\n";  
    echo $err;  
}  
$old_except_handle = set_exception_handler("exceptHandle");  
//$t = I_AM_NOT_DEFINED;    //generates a warning  
$a;  
throw new Exception('Uncaught Exception occurred');      
?>  
~~~

#### register_shutdown_function
 
执行机制是：php把要调用的函数调入内存。当页面所有ＰＨＰ语句都执行完成时，再调用此函数。
一般与trigger_error("...", E_USER_ERROR)，配合使用。

~~~php
<?php  
error_reporting(0);  
date_default_timezone_set('Asia/Shanghai');  
register_shutdown_function('my_exception_handler');  
  
$t = I_AM_NOT_DEFINED;  //generates a warning  
trigger_error("Vectors need to be of the same size", E_USER_ERROR);       
  
function my_exception_handler()  
{  
    if($e = error_get_last()) {  
    //$e['type']对应php_error常量  
    $message = '';  
    $message .= "出错信息:\t".$e['message']."\n\n";  
    $message .= "出错文件:\t".$e['file']."\n\n";  
    $message .= "出错行数:\t".$e['line']."\n\n";  
    $message .= "\t\t请工程师检查出现程序".$e['file']."出现错误的原因\n";  
    $message .= "\t\t希望能您早点解决故障出现的原因<br/>";  
    echo $message;  
    //sendemail to  
    }  
}  
?>  
~~~

### 简要说明错误处理：

#### 使用指定的文件记录错误报告日志

使用指定的文件记录错误报告日志使用指定的文件记录错误报告日志使用指定的文件记录错误报告日志 如果使用自己指定的文件记录错误日志，一定要确保将这个文件存放在文档根目录之外，以减少遭到攻击的可能。并且该文件一定要让PHP脚本的执行用户（Web服务器进程所有者）具有写权限。假设在Linux操作系统中，将/usr/local/目录下的error.log文件作为错误日志文件，并设置Web服务器进程用户具有写的权限。然后在PHP的配置文件中，将error_log指令的值设置为这个错误日志文件的绝对路径。

```
需要将php.ini中的配置指令做如下修改： 
error_reporting  =  E_ALL                   ;将会向PHP报告发生的每个错误     
display_errors = Off                        ;不显示满足上条 指令所定义规则的所有错误报告     
log_errors = On                             ;决定日志语句记录的位置     
log_errors_max_len = 1024                   ;设置每个日志项的最大长度     
error_log = /usr/local/error.log                ;指定产生的 错误报告写入的日志文件位置    
```

PHP的配置文件按上面的方式设置完成以后，并重新启动Web服务器。这样，在执行PHP的任何脚本文件时，所产生的所有错误报告都不会在浏览器中显示，而会记录在自己指定的错误日志/usr/local/error.log中。此外，不仅可以记录满足error_reporting所定义规则的所有错误，而且还可以使用PHP中的error_log()函数，送出一个用户自定义的错误信息。
该函数的原型如下所示：

      1. bool error_log ( string message [, int message_type  [, string destination [, string extra_headers]]] ) 
 
此函数会送出错误信息到Web服务器的错误日志文件、某个TCP服务器或到指定文件中。该函数执行成功则返回TRUE，失败则返回FALSE。第一个参数message 是必选项，即为要送出的错误信息。如果仅使用这一个参数，会按配置文件php.ini中所设置的位置处发送消息。第二个参数message_type为整数值：0表示送到操作系统的日志中；1则使用PHP的Mail()函数，发送信息到某E-mail处，第四个参数extra_headers亦会用到；2则将错误信息送到TCP 服务器中，此时第三个参数destination表示目的地IP及Port；3则将信息存到文件destination中。

如果以登入Oracle数据库出现问题的处理为例，该函数的使用如下所示： 

~~~php
<?php        
    if(!Ora_Logon($username, $password)){       
          error_log("Oracle数据库不可用!", 0);        //将错误消息写入到操作系统日志中     
     }     
    if(!($foo=allocate_new_foo()){     
         error_log("出现大麻烦了!", 1, ". mydomain.com");   //发送到管理员邮箱中     
    }    
     error_log("搞砸了!",   2,   "localhost:5000");     //发送到本机对应5000端口的服务器中     
     error_log("搞砸了!",   3,   "/usr/local/errors.log");  //发送到指定的文件中     
?>    
~~~

#### 错误信息记录到操作系统的日志里

错误信息记录到操作系统的日志里错误信息记录到操作系统的日志里错误信息记录到操作系统的日志里 错误报告也可以被记录到操作系统日志里，但不同的操作系统之间的日志管理有点区别。在Linux上错误语句将送往syslog，而在Windows上错误将发送到事件日志里。如果你不熟悉syslog，起码要知道它是基于UNIX的日志工具，它提供了一个API来记录与系统和应用程序执行有关的消息。Windows事件日志实际上与UNIX的syslog相同，这些日志通常可以通过事件查看器来查看。如果希望将错误报告写到操作系统的日志里，可以在配置文件中将error_log指令的值设置为syslog。

具体需要在php.ini中修改的配置指令如下所示： 
```
error_reporting  =  E_ALL                   ;将会向PHP报告发生的每个错误     
display_errors = Off                            ;不显示 满足上条指令所定义规则的所有错误报告     
log_errors = On                             ;决定日志语句记录的位置     
log_errors_max_len = 1024                   ;设置每个日志项的最大长度     
error_log = syslog                          ;指定产生的错误报告写入操作系统的日志里    
```

除了一般的错误输出之外，PHP还允许向系统syslog中发送定制的消息。虽然通过前面介绍的error_log()函数，也可以向syslog中发送定制的消息，但在PHP中为这个特性提供了需要一起使用的4个专用函数。
分别介绍如下：

- define_syslog_variables() 

在使用openlog()、syslog及closelog()三个函数之前必须先调用该函数。因为在调用该函数时，它会根据现在的系统环境为下面三个函数初使用化一些必需的常量。 

- openlog() 

打开一个和当前系统中日志器的连接，为向系统插入日志消息做好准备。并将提供的第一个字符串参数插入到每个日志消息中，该函数还需要指定两个将在日志上下文使用的参数，可以参考官方文档使用。
 
- syslog()

该函数向系统日志中发送一个定制消息。需要两个必选参数，第一个参数通过指定一个常量定制消息的优先级。例如LOG_WARNING表示一般的警告，LOG_EMERG表示严重地可以预示着系统崩溃的问题，一些其他的表示严重程度的常量可以参考官方文档使用。第二个参数则是向系统日志中发送的定制消息，需要提供一个消息字符串，也可以是PHP引擎在运行时提供的错误字符串。 

- closelog()

该函数在向系统日志中发送完成定制消息以后调用，关闭由openlog()函数打开的日志连接。 
 
如果在配置文件中，已经开启向syslog发送定制消息的指令，就可以使用前面介绍的四个函数发送一个警告消息到系统日志中，并通过系统中的syslog解析工具，查看和分析由PHP程序发送的定制消息，如下所示： 

~~~php
<?php  
      define_syslog_variables();     
     openlog("PHP5", LOG_PID , LOG_USER);     
     syslog(LOG_WARNING, "警告报告向syslog中发送的演示， 警告时间：".date("Y/m/d H:i:s"));    
     closelog();  
~~~ 

以Windows系统为例，通过右击"我的电脑"选择管理选项，然后到系统工具菜单中，选择事件查看器，再找到应用程序选项，就可以看到我们自己定制的警告消息了。上面这段代码将在系统的syslog文件中，生成类似下面的一条信息，是事件的一部分： 

1. PHP5[3084], 警告报告向syslog中发送的演示， 警告时间：2009/03/26 04:09:11.  

使用指定的文件还是使用syslog记录错误日志，取决于你所在的Web服务器环境。如果你可以控制Web服务器，使用syslog是最理想的，因为你能利用syslog的解析工具来查看和分析日志。但如果你的网站在共享服务器的虚拟主机中运行，就只有使用单独的文本文件记录错误日志了。

本文引自[这里](http://blog.csdn.net/hguisu/article/details/7464977)