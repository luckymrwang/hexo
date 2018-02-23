title: SSH自动登录
date: 2017-11-28 10:05:25
tags: [Linux]
toc: true
---
```
#!/usr/bin/expect
spawn ssh ubuntu@192.168.1.1 -p 22
expect "*password:"
send "xxxx\r"
send "su ubuntu\r"
expect "*Password:"
send "su ubuntu\r"
expect "*Password:"
send "xxxx\r"
send "cd /data/PC\r"
send "./g.sh\r"
expect "*#"
interact
```
<!-- more -->

### Expect中最关键的四个命令是send,expect,spawn,interact。

- send：用于向进程发送字符串
- expect：从进程接收字符串
- spawn：启动新的进程
- interact：允许用户交互

#### send 命令

send命令接收一个字符串参数，并将该参数发送到进程。

```
expect1.1> send "hello world\n"
hello world
```

#### expect 命令

expect命令和send命令正好相反，expect通常是用来等待一个进程的反馈。expect可以接收一个字符串参数，也可以接收正则表达式参数。和上文的send命令结合，现在我们可以看一个最简单的交互式的例子：

```
expect "hi\n"
send "hello there!\n"
```

这两行代码的意思是：从标准输入中等到hi和换行键后，向标准输出输出hello there。

*tips： $expect_out(buffer)存储了所有对expect的输入，<$expect_out(0,string)>存储了匹配到expect参数的输入。*

比如如下程序：

```
expect "hi\n"
send "you typed <$expect_out(buffer)>"
send "but I only expected <$expect_out(0,string)>"
```

当在标准输入中输入

```
test
hi
```

是，运行结果如下

```
you typed: test
hi
I only expect: hi
```

#### 模式-动作

expect最常用的语法是来自tcl语言的模式-动作。这种语法极其灵活，下面我们就各种语法分别说明。

单一分支模式语法：

expect "hi" {send "You said hi"}

匹配到hi后，会输出"you said hi"

多分支模式语法：

expect "hi" { send "You said hi\n" } \
"hello" { send "Hello yourself\n" } \
"bye" { send "That was unexpected\n" }

匹配到hi,hello,bye任意一个字符串时，执行相应的输出。等同于如下写法：

```
expect {
"hi" { send "You said hi\n"}
"hello" { send "Hello yourself\n"}
"bye" { send "That was unexpected\n"}
}
```

#### spawn 命令

上文的所有demo都是和标准输入输出进行交互，但是我们跟希望他可以和某一个进程进行交互。spawm命令就是用来启动新的进程的。spawn后的send和expect命令都是和spawn打开的进程进行交互的。结合上文的send和expect命令我们可以看一下更复杂的程序段了。

```
set timeout -1
spawn ftp ftp.test.com      //打开新的进程，该进程用户连接远程ftp服务器
expect "Name"             //进程返回Name时
send "user\r"        //向进程输入anonymous\r
expect "Password:"        //进程返回Password:时
send "123456\r"    //向进程输入don@libes.com\r
expect "ftp> "            //进程返回ftp>时
send "binary\r"           //向进程输入binary\r
expect "ftp> "            //进程返回ftp>时
send "get test.tar.gz\r"  //向进程输入get test.tar.gz\r
```

这段代码的作用是登录到ftp服务器ftp ftp.uu.net上，并以二进制的方式下载服务器上的文件test.tar.gz。

#### interact

到现在为止，我们已经可以结合spawn、expect、send自动化的完成很多任务了。但是，如何让人在适当的时候干预这个过程了。比如下载完ftp文件时，仍然可以停留在ftp命令行状态，以便手动的执行后续命令。interact可以达到这些目的。下面的demo在自动登录ftp后，允许用户交互。

```
spawn ftp ftp.test.com
expect "Name"
send "user\r"
expect "Password:"
send "123456\r"
interact
```

下面一段脚本实现了从机器A登录到机器B，然后执行机器B上的pwd命令，并停留在B机器上，等待用户交互。具体含义请参考上文。

```
 #!/home/tools/bin/64/expect -f
 set timeout -1  
 spawn ssh $BUser@$BHost
 expect  "*password:" { send "$password\r" }
 expect  "$*" { send "pwd\r" }
 interact
```

本文引自[这里](https://www.cnblogs.com/lzrabbit/p/4298794.html)

