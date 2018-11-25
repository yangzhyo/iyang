---
layout: post
title: "PHP 调试利器 - Xdebug"
date: 2017-06-10 06:21:00
categories: PHP
tags: [PHP,Xdebug]
---

> 工欲善其事必先利其器，学习一门新语言，调试器的使用必不可少，我们一般习惯通过 echo 或者 var_dump 来调试程序，但还有更多的选择。

<!-- more -->

PHP 目前有两种流行的调试器，

* [Xdebug](https://xdebug.org/)
* [Zend Debugger](https://www.zend.com/en/products/server/z-ray)

我在学习和研究过程中，发现网上的各种信息过于凌乱，感到痛苦，影响理解和使用。今天我以 Xdebug 为例，先解释其基本的工作原理，再说明生产过程中如何使用，希望能带给大家清晰易懂的知识。

# Xdebug

Xdebug 是一个 PHP 扩展，它采用 DBGp 协议，提供了对 PHP 进行 Debugging 和 Profiling 的能力。

Debug 信息包含：

* stack and function traces in error messages with:
	1. full parameter display for user defined functions
	2. function name, file name and line indications
	3. support for member functions
* memory allocation
* protection for infinite recursions

此外，Xdebug 还提供以下功能：

* profiling information for PHP scripts
* code coverage analysis
* capabilities to debug your scripts interactively with a debugger front-end

今天只讨论 Debug，其它功能以后有机会再分享。

## Debug原理

原文参考：[https://xdebug.org/docs/remote](https://xdebug.org/docs/remote)

简单来说，XDebug 调试是一种 C/S 结构，Client 是 PHP-Xdebug 插件，Server 是我们的 IDE（或者各种Editor插件），中间通过 [DBGp](https://xdebug.org/docs-dbgp.php) 协议通信。PHP 脚本在运行时，由 Xdebug 插件向 IDE 发起连接，将调试信息发送给 IDE，并通过 DBGp 协议进行互动。

看下图，就明白了：

![xdebug运行结构](https://xdebug.org/images/docs/dbgp-setup.gif)

* The IP of the server is 10.0.1.2 with HTTP on port 80
* The IDE is on IP 10.0.1.42, so [xdebug.remote_host](https://xdebug.org/docs/all_settings#remote_host) is set to 10.0.1.42
* The IDE listens on port 9000, so [xdebug.remote_port](https://xdebug.org/docs/all_settings#remote_port) is set to 9000
* The HTTP request is started on the machine running the IDE
* Xdebug connects to 10.0.1.42:9000
* Debugging runs, HTTP Response provided

以上是单人模式，在 php.ini 中配置了 IDE 的 IP 地址以及监听的端口。这种模式的缺陷是配死了 IDE 目标，不能支持多用户调试。所以 Xdebug 还提供了一种多用户的调试模式，这种模式无需配置 remote_host，而是配置 [xdebug.remote_connect_back](https://xdebug.org/docs/all_settings#remote_connect_back)=On，Xdebug 会记下来访地址，作为调试时的连接目标。调试过程如下：

![xdebug运行结构](https://xdebug.org/images/docs/dbgp-setup2.gif)

* The IP of the server is 10.0.1.2 with HTTP on port 80
* The IDE is on an unknown IP, so [xdebug.remote_connect_back](https://xdebug.org/docs/all_settings#remote_connect_back) is set to 1
* The IDE listens on port 9000, so [xdebug.remote_port](https://xdebug.org/docs/all_settings#remote_port) is set to 9000
* The HTTP request is made, Xdebug detects the IP addres from the HTTP headers
* Xdebug connects to the detected IP (10.0.1.42) on port 9000
* Debugging runs, HTTP Response provided

搞明白了原理，那我们就开始实践吧。

## 安装Xdebug

根据各自不同的工作站平台，参考[官方安装指导](https://xdebug.org/docs/install).

安装的时候注意一下输出信息，记下安装好的扩展包位置，下一步配置PHP时会用到。

## 配置PHP-Xdebug

打开 php.ini（不知道 php.ini 在哪里的，可以输出 phpinfo() 看看），然后加上对 Xdebug 扩展的配置：

```shell
[Xdebug]
zend_extension="/usr/lib/php/extensions/no-debug-non-zts-20131226/xdebug.so"
xdebug.remote_enable=On
```
xdebug.remote_enable=On，表示打开远程调试开关，这是必须的。

然后，有两种方法可以验证一下插件是否加载成功：

1. 命令行执行 php -v，看看

	![pic](http://imgur.com/3UbEnNo.png)
2. 网页输出 phpinfo()

	![pic](http://imgur.com/r3I8FZF.png)
	
以上输出若能看到 Xdebug 版本信息，就 ok 了。另外，在 phpinfo 中，可以看到 xdebug 的配置信息，搜搜看，这些配置都是可以在 php.ini 中进行修改的，各自代表的意思大家可以查官网，这里就不详述了。

![phpinfo](http://imgur.com/mYvST9x.png)

## 配置IDE
以 PHPStorm 为例，PHPStorm 实现了 DBGp 协议，我们需要配置它监听的端口，这个端口号需要与 xdebug.remote_port 一致，才能确保调试时 PHP-Xdebug 能连上 IDE。

以下是默认配置，如果没有冲突可以不用改。在帮其他同学 trouble shooting 时，发现 php-fpm 和 xdebug 默认都使用 9000 号端口，真是一个奇怪的配置啊，难道他们俩不应该经常被同时使用吗？如果你发现自己的 debug 不能正常工作，不妨检查一下这一点，将他们调整为不同的端口。

![pic](http://imgur.com/LO1jY1d.png)

## 开始Debug
之前说过，IDE 是作为一个 Server 的角色监听特定端口，等待 Xdebug 来连接，对于 PHPStorm 开启监听非常简单，直接点击右上角的“小电话”按钮：

![pic](http://imgur.com/AkOjV7B.png)

变成

![pic](http://imgur.com/flNtKQS.png)

然后在代码里面打下断点。

接下来，开始运行并调试代码。针对不同类型的应用，方法也不一样。但目的都是相同的，就是要告诉 PHP-Xdebug，我要开始调试了，给我把调试信息发过来！

### Web App
两种方法：

1. 在请求的 URL 地址后面加一个 QueryString:XDEBUG_SESSION_START=session_name（当然用 POST 参数也可以），这样 PHP-Xdebug 就会知道该连接 IDE 调试了。

	其实 IDE 提供了快捷的操作方式，不用你手动写 URL 参数。见 [Debugging PHP Web Applications with Run Debug Configurations](https://confluence.jetbrains.com/display/PhpStorm/Debugging+PHP+Web+Applications+with+Run+Debug+Configurations)，简单来说，就是你需要配置好 Web Server，PHP Web Application，再点击后上角的“小瓢虫”按钮，IDE 就会自动打开浏览器，帮你输好网址，并添加 QueryString:XDEBUG_SESSION_START=session_name。
	
2. 安装浏览器插件，插件会在你请求一个 PHP 页面时，带上一个 XDEBUG_SESSION Cookie，这样就不用在 URL 上带参数了。
	
	支持的插件：
	* Firefox: [the-easiest-xdebug](https://addons.mozilla.org/en-US/firefox/addon/the-easiest-xdebug/)
	* Chrome: [Xdebug helper](https://chrome.google.com/extensions/detail/eadndfjplgieldjbigjakmdgkmoaaaoc)
	* Safari: [xdebug-toggler-for-safari](http://benmatselby.posterous.com/xdebug-toggler-for-safari)
	* Opera: [xdebug-launcher](https://addons.opera.com/addons/extensions/details/xdebug-launcher/?display=en)

试试以上两种方法，断点命中！开始享受调试。

### Console

一般用在命令行程序或者单元测试的调试中。

我们需要通过设置环境变量 XDEBUG_CONFIG 来通知 Xdebug 进行调试。当然，我们可以向这个环境变量中添加更多的设置来配置 Xdebug（这些配置都可以在 php.ini 中进行设置）。

```shell
export XDEBUG_CONFIG="idekey=session_name"
php myscript.php
```
	
断点命中！
	
不想调试了，释放掉这个环境变量。
	
```shell
unset XDEBUG_CONFIG
```
	
小技巧，如果你用 Zsh，可以为上述两段小代码设置别名。
	
```shell
alias xdebug-on="export XDEBUG_CONFIG=\"idekey=PHPSTORM\""
alias xdebug-off="unset XDEBUG_CONFIG"
```
	
PHPStorm 也提供了快捷的操作，见 [Debugging PHP CLI scripts with PhpStorm](https://confluence.jetbrains.com/display/PhpStorm/Debugging+PHP+CLI+scripts+with+PhpStorm)
	
# 结束语
搞完上述研究，真是够折腾，真心觉得还是怀念 VS，不愧是宇宙第一 IDE，啥都不用管。总之，希望这篇小文章能让你感到清晰一点，有问题或者建议，可以留下评论一起讨论。
