title: 解决 Homebrew 下载更新极慢的问题
date: 2021-01-28 14:42:30
tags: [Mac]
---

## 症状

使用 Homebrew 安装软件的时候一直卡在 Update 阶段。同时发现从 [github.com](http://github.com/) 下载文件也极度缓慢（几十 KB/s）

## 问题定位

使用 `brew update --verbose` 观察 update 过程：

<!-- more -->
```sh
brew update --verbose
Checking if we need to fetch /usr/local/Homebrew...
Checking if we need to fetch /usr/local/Homebrew/Library/Taps/caskroom/homebrew-fonts...
Fetching /usr/local/Homebrew...
Checking if we need to fetch /usr/local/Homebrew/Library/Taps/homebrew/homebrew-cask...
Checking if we need to fetch /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core...
Fetching /usr/local/Homebrew/Library/Taps/homebrew/homebrew-cask...
Fetching /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core...
remote: Enumerating objects: 337, done.
remote: Counting objects: 100% (337/337), done.
remote: Compressing objects: 100% (88/88), done.
remote: Total 298 (delta 221), reused 287 (delta 210), pack-reused 0
Receiving objects: 100% (298/298), 50.91 KiB | 39.00 KiB/s, done.
Resolving deltas: 100% (221/221), completed with 39 local objects.
From https://github.com/Homebrew/homebrew-core
   65a45a9..583b7f1  master     -> origin/master
remote: Enumerating objects: 179429, done.
remote: Counting objects: 100% (179429/179429), done.
remote: Compressing objects: 100% (56607/56607), done.
Receiving objects:   4% (7628/177189), 1.48 MiB | 8.00 KiB/s
```

发现 update 卡在从 github 仓库获取文件的过程。这个结果与手动从 github 下载文件慢的症状相互印证

## 解决

由于问题主要是在国内网络环境 github 下载慢，因此尝试：

- 更换使用国内的 homebrew 镜像源
- 使用代理访问 github.com

## 更换 Homebrew 源

使用以下命令更换国内阿里云上的 homebrew 镜像：

```sh
# 替换 brew.git:
cd "$(brew --repo)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git
# 替换 homebrew-core.git:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git

# 替换 homebrew-bottles:
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.zshrc
source ~/.zshrc
```

替换后，问题依旧，继续查看日志：

```sh
brew update --verbose
Checking if we need to fetch /usr/local/Homebrew...
Checking if we need to fetch /usr/local/Homebrew/Library/Taps/caskroom/homebrew-fonts...
Fetching /usr/local/Homebrew...
Checking if we need to fetch /usr/local/Homebrew/Library/Taps/homebrew/homebrew-cask...
Checking if we need to fetch /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core...
Fetching /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core...
From https://mirrors.aliyun.com/homebrew/homebrew-core
 + 583b7f1...8435590 master     -> origin/master  (forced update)
Fetching /usr/local/Homebrew/Library/Taps/homebrew/homebrew-cask...
Fetching /usr/local/Homebrew/Library/Taps/caskroom/homebrew-fonts...
remote: Enumerating objects: 179429, done.
remote: Counting objects: 100% (179429/179429), done.
remote: Compressing objects: 100% (56607/56607), done.
Receiving objects:   6% (11170/177189), 2.16 MiB | 30.00 KiB/s
```

可以看到由于homebrew-cask的仓库依然指向了 Github，这个过程还是慢。阿里云的 [镜像站](https://developer.aliyun.com/mirror/) 没有提供homebrew-cask，进一步搜索找到 [USTC 镜像站](http://mirrors.ustc.edu.cn/)，该站提供了homebrew-cask的源。使用上述同样的命令更换源：

```sh
# 替换 homebrew-cask.git:
cd "$(brew --repo)"/Library/Taps/homebrew/homebrew-cask
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git
```

测试发现问题解决。

## 恢复默认配置

首先执行下述命令:

```sh
# 重置brew.git:
	$ cd "$(brew --repo)"
	$ git remote set-url origin https://github.com/Homebrew/brew.git
	# 重置homebrew-core.git:
	$ cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
	$ git remote set-url origin https://github.com/Homebrew/homebrew-core.git
```

然后删掉 HOMEBREW_BOTTLE_DOMAIN 环境变量，将终端文件 `~/.bash_profile` 或者 `~/.zshrc` 中的 `HOMEBREW_BOTTLE_DOMAIN` 删掉，并执行

```sh
source ~/.bash_profile

或者

source ~/.zshrc
```
 

>官方源地址： 

>[https://github.com/Homebrew/brew.git](https://github.com/Homebrew/brew.git)

>[https://github.com/Homebrew/homebrew-core.git](https://github.com/Homebrew/homebrew-core.git)

>[https://github.com/Homebrew/homebrew-cask](https://github.com/Homebrew/homebrew-core.git)
