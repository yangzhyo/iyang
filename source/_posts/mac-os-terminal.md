---
layout: post
title: "折腾 macOS 的 Terminal 环境"
date: 2017-05-30 10:25:00
categories: 折腾
tags: [macOS]
---

> 今天是传统端午节，大家吃粽子了吗？老羊在上海祝大家节日快乐。

最近老羊的工作语言从 .NET 换成了 PHP，工作站也从 Windows 切到了 macOS 上。新的环境，必然是各种折（Zhuang）腾（Bi），今天先折腾 Terminal，默认的实在是有点古董。

<!-- more -->

# 改装 macOS 默认 Terminal

先看看效果：

![my terminal](http://i.imgur.com/SbA1VGI.png)

▼ 改装清单
* iTerm 2
* iTerm Solarized theme
* zsh
* oh-my-zsh
* Powerline font
* zsh agnoster theme
* zsh-syntax-highlighting plugin

# 改装过程

## iTerm
> **[iTerm](http://www.iterm2.com/)** 是 macOS 默认终端程序（Terminal）的替代品，它带来更现代的体验。

### 如何安装？

直接 [下载](http://www.iterm2.com/downloads.html)，拖拽到应用 `应用程序` 目录。

### 更改配色方案为 Solarized

iTerm2 内置了几套配色方案，大家可以根据自己的喜好选择。直接在 ```iTerm2 - Preferences - Profiles - Colors``` 里面配置就好。在下用的是 Solarized Dark.

![Colors](http://imgur.com/U5P9pBD.png)

### 彩色的文字 
去掉 ```iTerm2 - Preferences - Profiles - Text - Text Rendering```，将 Draw bold text in bright colors 前面的勾，终端上该显示的彩色文字就回来了。

### 自动完成
通过快捷键 ```command+;``` 可自动联想出一些命令；
![AutoComp](http://www.iterm2.com/img/screenshots/autocomplete.png)

### 快捷呼出
iTerm 提供了非常方便快捷的呼出功能，不管你的桌面环境有多么复杂，一个快捷键就能把终端呼出。
在```iTerm2 - Preferences - Key```设置：

![Hotkey](http://imgur.com/PPgeHNI.png)

* 第一个选项是设置快捷呼出的快捷键；
* 第二个选项是另外一种形式的呼出（会在屏幕的上部展开一层半透明的窗体），如果这个选项未打开，默认是呼出 iTerm 窗体。
	![Hotkey](http://imgur.com/8Lg58dX.png)
	
	是不是很 Cool？注意这里可以为这个折叠层的显示设置单独设定一个 Profile，当然也可以共享默认的 Profile。

### 其它很酷的特性
这里不一一列举了，待大家到 [官网](http://www.iterm2.com/features.html) 发掘。

## zsh
macOS 默认 Shell 是 bash，而我们要改装的是另一款强大的 Shell: [zsh](http://www.zsh.org/)，全称 Z Shell，相比 bash 有大量的改进，参考[Wiki](https://zh.wikipedia.org/wiki/Z_shell)。

在 ```/etc/shells``` 文件列举了所有支持的 shell 解释器:

``` shell
cat /etc/shells
# List of acceptable shells for chpass(1).
# Ftpd will not allow users to connect who are not using
# one of these shells.

/bin/bash
/bin/csh
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
```
如果想改变默认的 shell，可执行 ```chsh -s /bin/zsh```。这里我们不用手动改变，安装好 oh-my-zsh 后会自动修改。

## oh-my-zsh
zsh 配置复杂，交给 [oh-my-zsh](http://ohmyz.sh/) 来管理。
> Oh-My-Zsh is an open source, community-driven framework for managing your ZSH configuration. It comes bundled with a ton of helpful functions, helpers, plugins, themes, and a few things that make you shout...

### 安装
``` shell
$ sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

### Agnoster theme
[Agnoster theme](https://github.com/agnoster/agnoster-zsh-theme) 是 zsh 的一个 theme，看起来非常的酷。

![agnoster-theme](https://gist.githubusercontent.com/agnoster/3712874/raw/screenshot.png)

要用好这套 theme，有两个准备条件：

1. 推荐配合 iTerm 2 + Solarized Dark 使用
	
	看默认的 Terminal + bash 有多丑：
	![terminal](http://imgur.com/c5CJGVB.png)
2. 安装 Powerline 字体，否则那些酷酷的符号无法正确显示
	
	这里下载：[Powerline-patched font](https://github.com/powerline/fonts)
	
	安装后，在 ```iTerm2 - Preferences - Profiles - Text - Font``` 修改字体为 Powerline 提供的字体，在下用的是 ```Meslo LG S DZ Regular for Powerline```

准备好上述条件后，

1. 从 Agnoster 的 [Repo](https://github.com/agnoster/agnoster-zsh-theme) 下载 Theme 到 ```~/.oh-my-zsh/themes/```（有可能皮肤已经默认被安装了，检查一下）
2. 修改配置文件 ```~/.zshrc```

	``` shell
   ZSH_THEME="agnoster"
	```
3. 命令行默认会显示用户名和机器名，可能会比较长，可以通过修改主题脚本来解决，将 ```~/.oh-my-zsh/themes/agnoster.zsh-theme``` 中的 ```prompt_conext()``` 方法中的代码注释掉即可。

## zsh-syntax-highlighting plugin
> 一个锦上添花的[插件](https://github.com/zsh-users/zsh-syntax-highlighting)，可以在你敲命令的时候，识别命令的正确性，正确显示为绿色，错误显示为红色。

### 安装过程

1. Clone this repository in oh-my-zsh's plugins directory:

	```git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting```
	
2. Activate the plugin in ~/.zshrc:
	
	```plugins=( [plugins...] zsh-syntax-highlighting)```
	
3. Source ~/.zshrc to take changes into account:
	
	```source ~/.zshrc```
	
### 更多很酷的插件
见 [awesome-zsh-plugins](https://github.com/unixorn/awesome-zsh-plugins)
