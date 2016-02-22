title: '[PHP]全局变量：global与��区别和使用'
date: 2016-01-08 23:11:01
tags: [PHP]
---

今天在写框架的时候想把SaeMySQL初始化之后作为全局变量使用。
但是后来发现PHP中的全局变量和Java或者OC中的全局变量还是有较大区别的。
下面记录一下php里面的global的使用相关注意事项。
1.有些场合需要全局变量的出现，如下例子：

<!-- more -->
```php
<?php
$name="why";//定义变量name,并初始化
function echoName()
{
//试图引用函数外面的变量
echo "myname is ".$name."<br>";
}
echoName();
?>
```

上面的代码的结果为："myname is" 。而不是期望中的："myname is why"。因为函数没有传递参数$name的值，企图引用外部变量，不会成功。这时候考虑使用global。

2.于是将上述代码改为

```php
<?php
global $name="why";//用global声明的同时赋值
function echoName()
{
//试图引用函数外面的变量
echo "myname is ".$name."<br>";
}
echoName();
?>
```

结果为：Parse error: syntax error, unexpected '=', expecting ',' or ';' in http:\\xxxxxxx.com on line 2
也即上述代码有错误。原因是不能在用global声明变量的同时给变量赋值。

3.再次更改上述代码：

```php
<?php
global $name;
$name="why";//将global声明与赋值分开
function echoName()
{
//试图引用函数外面的变量
echo "myname is ".$name."<br>";
}
echoName();
?>
```

但是得到的结果依然为："myname is" ,原因是global的用法不对。

global的正确用法是："在一个函数中引入外部的一个变量，如果该变量没有通过参数传递进来，那么就通过global引入进来。" 也就是说，当一个函数引用一个外部变量时，可以在函数内通过global来声明该变量，这样该变量就可以在函数中使用了（相当于当作参数传递进来）。

4.于是进一步改动上述代码：

```php
<?php
$name="why";//定义变量name,并初始化
function echoName()
{
//通过global来声明$name，相当于传递参数
global $name;
echo "myname is ".$name."<br>";
}
echoName();
?>
```

此时得到期望中的结果："myname is why"。
以上代码说明，global是起传递参数的作用，而并非使变量的作用域为全局。

5.以下代码证明了这一点:

```php
<?php
$name="why";//声明变量$name,并初始化
function echoName1()
{
//在函数echoName1()里使用global来声明$name
global  $name;
echo "the first name is ".$name."<br>";
}
function echoName2()
{
//在函数echoName2()里没有使用global来声明$name
echo "the second name is ".$name."<br>";
}
echoName1();
echoName2();
?>
```

结果为:

the first name is why
the second name is

上面的结果说明在函数echoName2()中，$name变量仍然是未知的，因为没有用global来声明，也就没有传递进去。同时也证明了global的作用并不是使变量的作用域为全局。

综上，global的作用就相当于传递参数，在函数外部声明的变量，如果在函数内想要使用，就用global来声明该变量，这样就相当于把该变量传递进来了，就可以引用该变量了。

当然，除了通过上述方法外，还可以使用全局数组$GLOBALS来解决问题，在需要用到外部变量的地方，使用$GLOBALS['var']就可以了。例：

```php
<?php

$name="why";//定义变量name,并初始化
function echoName()
{
//通过全局数组$GLOBALS来引用外部变量
echo "myname is ".$GLOBALS['name']."<br>";
}
echoName();
?>
```

得到的结果为:   myname is why 。

文章内容引自[汪海的实验室](http://blog.csdn.net/pleasecallmewhy/article/details/8575492)
