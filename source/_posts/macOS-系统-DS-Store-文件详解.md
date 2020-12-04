title: macOS 系统 .DS_Store 文件详解
date: 2020-09-28 09:03:47
tags: [macOS]
---

### .DS_Store 文件

`.DS_Store` 是 `Finder` 用来存储这个文件夹的显示属性的：比如文件图标的摆放位置。

### 显示/隐藏 Mac 隐藏文件

- 显示：defaults write com.apple.finder AppleShowAllFiles -bool true
- 隐藏：defaults write com.apple.finder AppleShowAllFiles -bool false

<!-- more -->
### 删除 .DS_Store 文件

- find /path/to/files -name ".DS_Store" -delete
- find /path/to/files –type f –name ".DS_Store" -print –delete
- find /path/to/files –type f –name ".DS_Store" -print0 | xargs –0 rm -rdf

### 配置 SVN 忽略 .DS_Store 文件

1、编辑`~/.subversion/config`文件；
2、找到global-ignores配置项，取消注释；
3、添加上自己要忽略的文件，用空格隔开

```sh
global-ignores = *.iml .idea .DS_Store .sass-cache node_modules *.o *.lo *.la *.al .libs *.so *.so.[0-9]* *.a *.pyc *.pyo
```

这是针对客户端的全局修改，不会对 SVN 服务端有影响，忽略的文件列表不会再出现在 SVN 的操作中。

### 防止 .DS_Store 文件生成

```sh
defaults write com.apple.desktopservices DSDontWriteNetworkStorestrue true
```

### 配置 Git 忽略 .DS_Store 文件

- `.gitignore` 配置文件用于配置不需要加入版本管理的文件
- 语法
	- 以斜杠”/“开头表示目录；
	- 以星号”*“通配多个字符；
	- 以问号”?“通配单个字符
	- 以方括号”[]“包含单个字符的匹配列表；
	- 以叹号”!“表示不忽略(跟踪)匹配到的文件或目录；
- `Git` 对于 `.gitignore` 配置文件是按行从上到下进行规则匹配的，意味着如果前面的规则匹配的范围更大，则后面的规则将不会生效。

	1、对该 `repo` 的所有用户应用过滤 将 `.gitignore` 文件放在工作目录的跟目录，编辑 `.gitignore` 完成后提交 git add `.gitignore`

	2、仅对自己的 `repo` 备份过滤 添加/编辑你工作目录的`$GIT_DIR/info/exclude`，例如你的`working copy`目录是`~/src/project1`，则路径为`~/src/project1/.git/info/exclude`

	3、系统全局过滤 创建一个 `ignore` 文件，名字随意起，比如我的放在`~/.gitglobalignore`，然后配置 git：

```sh
git config —global core.excludesfile = ~/.gitglobalignore
```

- 忽略 `.DS_Store` 添加 `.DS_Store` 到 `.gitignore` 文件即可