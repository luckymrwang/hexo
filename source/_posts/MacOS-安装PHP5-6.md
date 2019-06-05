title: MacOS 安装PHP5.6
date: 2019-06-05 11:53:59
tags: [PHP]
---

`MacOS Mojave` 系统之后，如果想安装 `php5.6` 版本的时候，无法用`brew install php5.6` 安装，因为在新的 `brew` 中已经废弃了 `php5.6` 和 `php7.0`，如果使用 `brew search php` 搜索出来的Php版本最低是 php@7.1 的，所以有相关需求的可以按照下面方法安装

<!-- more -->
#### 添加源

```
brew tap exolnet/homebrew-deprecated
```

#### 搜索PHP

```
brew search php
```

#### 安装PHP

```
brew install php@5.6
```

安装完后会提示如下信息：

```
==> php@5.6
To enable PHP in Apache add the following to httpd.conf and restart Apache:
    LoadModule php5_module /usr/local/opt/php@5.6/lib/httpd/modules/libphp5.so
    <FilesMatch \.php$>
        SetHandler application/x-httpd-php
    </FilesMatch>
Finally, check DirectoryIndex includes index.php
    DirectoryIndex index.php index.html
The php.ini and php-fpm.ini file can be found in:
    /usr/local/etc/php/5.6/
php@5.6 is keg-only, which means it was not symlinked into /usr/local,
because this is an alternate version of another formula.
If you need to have php@5.6 first in your PATH run:
  echo 'export PATH="/usr/local/opt/php@5.6/bin:$PATH"' >> ~/.bash_profile
  echo 'export PATH="/usr/local/opt/php@5.6/sbin:$PATH"' >> ~/.bash_profile
For compilers to find php@5.6 you may need to set:
  export LDFLAGS="-L/usr/local/opt/php@5.6/lib"
  export CPPFLAGS="-I/usr/local/opt/php@5.6/include"

To have launchd start exolnet/deprecated/php@5.6 now and restart at login:
  brew services start exolnet/deprecated/php@5.6
Or, if you don't want/need a background service you can just run:
  sudo php-fpm
```

#### 配置环境变量

`vim ~/.bash_profile`

```
#php56
export PATH=/usr/local/Cellar/php56/5.6.40/bin:$PATH
#php56-fpm
export PATH=/usr/local/Cellar/php56/5.6.40/sbin:$PATH
```



