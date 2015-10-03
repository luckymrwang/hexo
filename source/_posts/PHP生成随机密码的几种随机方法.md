title: PHP生成随机密码的几种方法
date: 2015-07-22 16:31:10
tags: PHP
---

## 方法一：

 1、在 33 – 126 中生成一个随机整数，如 35

 2、将 35 转换成对应的ASCII码字符，如 35 对应 #

 3、重复以上 1、2 步骤 n 次，连接成 n 位的密码

该算法主要用到了两个函数，`mt_rand ( int $min , int $max )` 函数用于生成随机整数，其中 $min – $max 为 ASCII 码的范围，<!-- more -->这里取 33 -126 ，可以根据需要调整范围，如ASCII码表中 97 – 122 位对应 a – z 的英文字母，具体可参考 ASCII码表； `chr ( int $ascii )` 函数用于将对应整数 $ascii 转换成对应的字符

	function create_password($pw_length = 8)
	{
    	$randpwd = '';
    	for ($i = 0; $i < $pw_length; $i++) 
    	{
        	$randpwd .= chr(mt_rand(33, 126));
    	}
    	return $randpwd;
	}

	// 调用该函数，传递长度参数$pw_length = 6
	echo create_password(6);

## 方法二：

1、预置一个的字符串 $chars ，包括 a – z，A – Z，0 – 9，以及一些特殊字符

2、在 $chars 字符串中随机取一个字符

3、重复第二步 n 次，可得长度为 n 的密码

	function generate_password( $length = 8 ) {
    	// 密码字符集，可任意添加你需要的字符
    	$chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()-_ []{}<>~`+=,.;:/?|';

    	$password = '';
    	for ( $i = 0; $i < $length; $i++ ) 
    	{
        	// 这里提供两种字符获取方式
        	// 第一种是使用 substr 截取$chars中的任意一位字符；
        	// 第二种是取字符数组 $chars 的任意元素
        	// $password .= substr($chars, mt_rand(0, strlen($chars) - 1), 1);
        	$password .= $chars[ mt_rand(0, strlen($chars) - 1) ];
    	}

    	return $password;
	}

##方法三：

1、预置一个的字符数组 $chars ，包括 a – z，A – Z，0 – 9，以及一些特殊字符

2、通过`array_rand()`从数组 $chars 中随机选出 $length 个元素

3、根据已获取的键名数组 $keys，从数组 $chars 取出字符拼接字符串。该方法的缺点是相同的字符不会重复取。

	function make_password( $length = 8 )
	{
    	// 密码字符集，可任意添加你需要的字符
    	$chars = array('a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 
    	'i', 'j', 'k', 'l','m', 'n', 'o', 'p', 'q', 'r', 's', 
    	't', 'u', 'v', 'w', 'x', 'y','z', 'A', 'B', 'C', 'D', 
    	'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L','M', 'N', 'O', 
    	'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y','Z', 
    	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', '!', 
    	'@','#', '$', '%', '^', '&', '*', '(', ')', '-', '_', 
    	'[', ']', '{', '}', '<', '>', '~', '`', '+', '=', ',', 
    	'.', ';', ':', '/', '?', '|');

    	// 在 $chars 中随机取 $length 个数组元素键名
    	$keys = array_rand($chars, $length); 

    	$password = '';
    	for($i = 0; $i < $length; $i++)
    	{
        	// 将 $length 个数组元素连接成字符串
        	$password .= $chars[$keys[$i]];
    	}

    	return $password;
	}

##方法四：

 1、time() 获取当前的 Unix 时间戳

 2、将第一步获取的时间戳进行 md5() 加密

 3、将第二步加密的结果，截取 n 位即得想要的密码

	function get_password( $length = 8 ) 
	{
    	$str = substr(md5(time()), 0, 6);
    	return $str;
	}

##时间效率对比：

我们使用以下PHP代码，计算上面的 4 个随机密码生成函数生成 6 位密码的运行时间，进而对他们的时间效率进行一个简单的对比。

	<?php
	function getmicrotime()
	{
    	list($usec, $sec) = explode(" ",microtime());
    	return ((float)$usec + (float)$sec);
	}
 
	// 记录开始时间
	$time_start = getmicrotime();
    
	// 这里放要执行的PHP代码，如:
	// echo create_password(6);
 
	// 记录结束时间
	$time_end = getmicrotime();
	$time = $time_end - $time_start;

 	// 输出运行总时间 
	echo "执行时间 $time seconds";
	?>

### 最终结果：

	 方法一：9.8943710327148E-5 秒
     方法二：9.6797943115234E-5 秒
     方法三：0.00017499923706055 秒
     方法四：3.4093856811523E-5 秒

*可以看出方法一和方法二的执行时间都差不多，方法四运行时间最短，而方法三的运行时间稍微长点。*