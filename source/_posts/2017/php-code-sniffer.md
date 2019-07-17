---
layout: post
title: "利用 Code Sniffer 检查 PHP 代码规范"
date: 2017-06-11 15:21:00
categories: 经验总结
tags: [PHP,Coding Standard]
---

> 作为一个新手，写出规范的代码也是一门必修课，除了阅读相应的代码规范文档之外，充分利用相关的工具能使得进阶之路事半功倍。今天这篇分享将简单地梳理一下 PHP 规范，并介绍一个代码检查工具 Code Sniffer，结合 PHPStorm 以及 GIT 进行实践。

<!-- more -->

# PHP 规范

为什么需要统一代码规范？

> PHP 社区百花齐放，拥有大量的函数库、框架和组件。PHP 开发者通常会在自己的项目中使用若干个外部库，因此 PHP 代码遵循（尽可能接近）同一个代码风格就非常重要，这让开发者可以轻松地将多个代码库整合到自己的项目中。

目前流行的一些规范：

* PSR
	
	PSR 是 PHP Standard Recommendations 的简写，由 [PHP FIG](https://github.com/php-fig) 组织制定的 PHP 规范，是 PHP 开发的实践标准。
	
	PHP FIG，FIG 是 Framework Interoperability Group（框架可互用性小组）的缩写，由几位开源框架的开发者成立于 2009 年，从那开始也选取了很多其他成员进来（包括但不限于 [Laravel](http://laravel.com/), [Joomla](https://www.joomla.org/), [Drupal](https://www.drupal.org/), [Composer](https://getcomposer.org/), [Phalcon](https://phalconphp.com/en/), [Slim](http://www.slimframework.com/), [Symfony](http://symfony.com/), [Zend Framework](http://framework.zend.com/) 等），虽然不是「官方」组织，但也代表了大部分的 PHP 社区。

	项目的目的在于：通过框架作者或者框架的代表之间讨论，以最低程度的限制，制定一个协作标准，各个框架遵循统一的编码规范，避免各家自行发展的风格阻碍了 PHP 的发展，解决这个程序设计师由来已久的困扰。

	目前已表决通过了 6 套标准，已经得到大部分 PHP 框架的支持和认可。
	
	详细介绍请参考官方说明：
	
	* [中文文档](https://psr.phphub.org/)
	* [官方网站](http://www.php-fig.org/psr/)

* PEAR

	The PEAR Coding Standards apply to code that is part of the official PEAR distribution. Coding standards often abbreviated as CS among developers and they aim to keep code consistent to be easily readable and maintainable by most of PEAR folks.
	
	参考 [官方说明](http://pear.php.net/manual/en/standards.php)
	
* Zend
	
	Zend Framework 使用的代码规范。
	
	参考 [官方说明](https://framework.zend.com/manual/1.11/en/coding-standard.html)

# Code Sniffer

官方网站：[https://github.com/squizlabs/PHP_CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer)

> PHP_CodeSniffer is a set of two PHP scripts; the main phpcs script that tokenizes PHP, JavaScript and CSS files to detect violations of a defined coding standard, and a second phpcbf script to automatically correct coding standard violations. PHP_CodeSniffer is an essential development tool that ensures your code remains clean and consistent.

简单来说，就是两个小工具，

* phpcs (PHP Code Standard)，用来检查代码规范
* phpcbf (PHP Code Beautifier and Fixer)，用来自动修复代码

## 安装 Code Sniffer

参考 [官方文档](https://github.com/squizlabs/PHP_CodeSniffer)

## 使用 Code Sniffer

**执行 phpcs 检查代码规范**

```shell
$ phpcs /path/to/code/myfile.php

FILE: /path/to/code/myfile.php
--------------------------------------------------------------------------------
FOUND 5 ERRORS AFFECTING 4 LINES
--------------------------------------------------------------------------------
 2 | ERROR | [ ] Missing file doc comment
 3 | ERROR | [x] TRUE, FALSE and NULL must be lowercase; expected "false" but
   |       |     found "FALSE"
 5 | ERROR | [x] Line indented incorrectly; expected at least 4 spaces, found 1
 8 | ERROR | [ ] Missing function doc comment
 8 | ERROR | [ ] Opening brace should be on a new line
--------------------------------------------------------------------------------
PHPCBF CAN FIX THE 2 MARKED SNIFF VIOLATIONS AUTOMATICALLY
--------------------------------------------------------------------------------
```

Code-Sniffer 默认使用 PEAR 代码规范。可以使用 `--standard=xxx` 指定。

```shell
$ phpcs --standard=PSR2 /path/to/code/myfile.php
```

或者修改默认配置

```shell
$ phpcs --config-set default_standard Squiz
```

**执行 phpcbs 修复代码**

```shell
$ phpcbf /path/to/code
Processing init.php [PHP => 7875 tokens in 960 lines]... DONE in 274ms (12 fixable violations)
    => Fixing file: 0/12 violations remaining [made 3 passes]... DONE in 412ms
Processing config.php [PHP => 8009 tokens in 957 lines]... DONE in 421ms (155 fixable violations)
    => Fixing file: 0/155 violations remaining [made 7 passes]... DONE in 937ms
Patched 2 files
Time: 2.55 secs, Memory: 25.00Mb
```

更多功能，参考 [官方文档](https://github.com/squizlabs/PHP_CodeSniffer/wiki)

## 与 PHP Storm 集成

将 Code Sniffer 与 PHP Storm 集成，可以使 IDE 实时显示代码的规范情况，提醒开发人员修改代码，长期使用，可以让开发人员养成非常规范的代码习惯。

![ide](https://a.iya.ng/images/blog/bl4kgYl.png)

如何配置？

1. 为 PHPStorm 配置 Code Sniffer 路径：
	![code-sniffer](https://a.iya.ng/images/blog/X4rIc0e.png)
	![code-sniffer](https://a.iya.ng/images/blog/TdqQVfz.png)
2. 设置 Inspections

	选择合适的 Coding Standard
	![](https://a.iya.ng/images/blog/hLmHbZo.png)
	
## 利用 GIT Hook 强制代码检查

使用 GIT 的 Pre-Commit Hook，可以在代码被提交之前，强制调用 Code Sniffer 进行代码检查，如果不符合规范，将阻止用户提交代码。

1. 在项目的 `.git/hooks` 建立 `pre-commit` 文件
2. 写入代码：
	[Pre-Commit Gist](https://gist.githubusercontent.com/yangzhyo/6dad52e862c5efa368c992efc52e1fb9/raw/dec18f55ddfdf6eeb6d659a19398ec2d106fface/pre-commit.php)
	
试试看:)

# 结束语

会使用工具进行生产活动是人类区别于动物成为智慧生物的本质，IT 技术生产活动亦是如此，使用工具提高生产效能则是一个优秀技术人员必备素质。
