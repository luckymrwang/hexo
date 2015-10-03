title: "在 Mac OS X 终端里使用 Solarized 配色方案"
date: 2015-06-04 11:27:50
tags: [Mac]
---

## 简介

[Solarized](http://ethanschoonover.com/solarized) 是目前最完整的 Terminal/Editor/IDE 配色项目，几乎覆盖所有主流操作系统（Mac OS X, Linux, Windows）、编辑器和 IDE（Vim, Emacs, Xcode, TextMate, NetBeans, Visual Studio 等），终端（iTerm2, Terminal.app, Putty 等）。类似的项目还有 [Tomorrow Theme](https://github.com/chriskempson/tomorrow-theme).
<!-- more -->
要在 Mac OS X 终端里舒服的使用命令行（至少）需要给3个工具配色，terminal、vim 和 ls. 首先下载 Solarized：

	$ git clone git://github.com/altercation/solarized.git

## Terminal/iTerm2

Mac OS X 自带的 Terminal 和免费的 [iTerm2](http://www.iterm2.com/) 都是很好用的工具，iTerm2 可以切分成多窗口，更方便一些。

如果你使用的是 Terminal 的话，在 solarized/osx-terminal.app-colors-solarized 下双击 Solarized Dark ansi.terminal 和 Solarized Light ansi.terminal 就会自动导入两种配色方案 Dark 和 Light 到 Terminal.app 里。

如果你使用的是 iTerm2 的话，到 solarized/iterm2-colors-solarized 下双击 Solarized Dark.itermcolors 和 Solarized Light.itermcolors 两个文件就可以把配置文件导入到 iTerm 里。

## Vim

Vim 的配色最好和终端的配色保持一致，不然在 Terminal/iTerm2 里使用命令行 Vim 会很别扭：

	$ cd solarized
	$ cd vim-colors-solarized/colors
	$ mkdir -p ~/.vim/colors
	$ cp solarized.vim ~/.vim/colors/

	$ vi ~/.vimrc
	syntax enable
	set background=dark
	colorscheme solarized

*上面是借鉴网上的配置，但我的 MacBook Pro 是如下配置的：*

	$ cd solarized
	$ cd vim-colors-solarized/colors
	$ cp solarized.vim /usr/share/vim/vim73/

	$ vim /usr/share/vim/vimrc
	syntax enable
	set background=dark

![vim](/images/solarized-vim.png)

## ls

Mac OS X 是基于 FreeBSD 的，所以一些工具 ls, top 等都是 BSD 那一套，ls 不是 GNU ls，所以即使 Terminal/iTerm2 配置了颜色，但是在 Mac 上敲入 ls 命令也不会显示高亮，可以通过安装 coreutils 来解决（brew install coreutils），不过如果对 ls 颜色不挑剔的话有个简单办法就是在 .bash_profile 里输出 CLICOLOR=1：

	$ vi ~/.bash_profile
	export CLICOLOR=1

![ls](/images/solarized-ls.png)
