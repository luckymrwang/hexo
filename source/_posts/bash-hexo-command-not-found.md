title: 'bash: hexo: command not found'
date: 2017-09-11 22:36:35
tags: [Hexo]
---

最近将`MacOS`升级到10.12.6，以及更新了`Homebrew`，结果发现之前正常的`hexo`命令不能使用了，于是更新了版本依旧如此。

折腾了很久后最终发现是`nvm`的原因

所以列出我的解决办法

- 安装 `brew install nvm`
- 创建文件目录 `mkdir ~/.nvm`
- 设置环境变量，将如下信息添加到`~/.bash_profile`中

```
export NVM_DIR="$HOME/.nvm"
. "/usr/local/opt/nvm/nvm.sh"
```

- `source ~/.bash_profile`
- 用`nvm`安装`Node.js` `nvm install stable`
- `npm install -g hexo-cli`
