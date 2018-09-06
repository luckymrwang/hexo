title: GitBook Build 导出html不跳转
date: 2018-09-06 11:29:18
tags: [GitBook]
---

新版本`gitbook build`后不支持本地页面内跳转

需要指定版本,另外node版本也需要6以下

```js
gitbook build --gitbook=2.6.7
```

<!-- more -->
如果遇到

```js
Error loading version latest: Error: Cannot find module 'internal/util/types'
```

需要将`node`版本降低

用`nvm`来管理

```
nvm ls-remote  #查看安装版本号
nvm install v6.14.4 #安装6版本
nvm use v6.14.4  #切换6
npm install npm -g  
npm install gitbook-cli -g
gitbook build --gitbook=2.6.7
nvm ls
nvm use v10.9.0  #切回
```

