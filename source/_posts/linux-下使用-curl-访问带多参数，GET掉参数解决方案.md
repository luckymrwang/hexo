title: linux 下使用 curl 访问带多参数，GET掉参数解决方案
date: 2016-11-15 18:16:15
tags: [Linux]
toc: true
---

url 为 `http://mywebsite.com/index.php?a=1&b=2&c=3`

web 形式下访问 url 地址，使用 \$_GET 是可以获取到所有的参数

`curl  -s  http://mywebsite.com/index.php?a=1&b=2&c=3`

然而在linux下，上面的例子 $_GET只能获取到参数 a

由于url中有 `&` 其他参数获取不到，在linux系统中 `&` 会使进程系统后台运行，必须对 `&` 进行下转义才能 $_GET 获取到所有参数

<!-- more -->

`curl  -s  http://mywebsite.com/index.php?a=1\&b=2\&c=3`

当然，最简单的方法 用双引号把整个url引起来就ok了

`curl  -s  "http://mywebsite.com/index.php?a=1&b=2&c=3"`

顺便再提一下 curl 中 post 传参数的方法

`curl  -d  'name=1&pagination=2' demoapp.sinap.com/worker.php`

这样 demoapp.sinap.com 站点中的 worker.php 脚本，就能得到 $_POST['name'] 和 $_POST['pagination'] 对应的值

再补充下curl获得网站信息的方法（ -s 表示静默  --head 表示取得head信息 ）

`curl  -s  --head  www.sina.com`

本文引自[这里](http://www.cnblogs.com/RodYang/p/3363296.html)