title: Sequel Pro SSH connection failed
date: 2016-08-08 18:11:30
tags: [Sequel Pro]
toc: true
---

Sequel Pro 通过 SSH 连接远程的MySQL报错

```
SSH connection failed, The SSH tunnel has unexpectedly closed.
```

<!-- more -->
产生问题的原因是这样的，被连接的远端我重新安装了服务器系统，然后又重装了MySQL，连接就出问题了。但是我用同样的参数用Navicate 就可以连接~

通过网上搜索有反映过这个问题，说是该软件的bug，然后升级到1.1.1版本就可以了，但是我现在使用的是1.1.2，早就应该解决了。

然后百试不通后我就换回了1.1.1版本，可是还是不行。

于是我就分析了它的报错的具体信息，其中提到是密钥对配对不正确，远端已经更新，本地也需要改变。于是打开本地`/Users/sino/.ssh/known_hosts`并将记录的远端的SSH-RSA删除，然后再次连接，这次终于连通了，问题解决了。