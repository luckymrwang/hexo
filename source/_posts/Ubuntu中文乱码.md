title: Ubuntu中文乱码
date: 2018-02-23 15:46:51
tags: [Linux]
toc: true
---

### 安装中文支持包
```
sudo apt-get install language-pack-zh-hans
```

### 在`/etc/environment`中追加：
```
LANG="zh_CN.UTF-8"
LANGUAGE="zh_CN:zh:en_US:en"
```

<!-- more -->
### 在`/var/lib/locales/supported.d/local`中追加
```
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
zh_CN.GBK GBK
zh_CN GB2312
```

### 执行

```
sudo locale-gen
```

### 对于中文乱码是空格的情况，安装中文字体
```
sudo apt-get install fonts-droid-fallback ttf-wqy-zenhei ttf-wqy-microhei fonts-arphic-ukai fonts-arphic-uming
```