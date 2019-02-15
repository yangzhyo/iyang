---
layout: post
title: "树莓派之重新折腾"
date: 2019-01-21 22:52:00
categories: 折腾
tags: [Raspberry Pi, Raspbian]
---

两年前从公司搞来一台树莓派，折腾一番就被扔进了箱子吃灰至今，见 [树莓派之初见](../2016/raspberry-pi-first-try.html)。最近又翻出来鼓捣了一番，这么小的板子就别装什么桌面版了，还得接个显示器、键盘、鼠标，搞成一台服务器用多好啊。

<!-- more -->

# 重刷系统

[Raspbian](https://www.raspbian.org/) 是一个运行在树莓派上的开源操作系统，它基于 [Debian](https://www.debian.org/) 进行了优化，并受到官方和社区的长期支持。

我需要抹掉之前安装的 Raspbian with Desktop，重新安装 Raspbian Lite，先把操作系统瘦身了。Lite 版只包含最基本的系统和软件，不包含图形化界面。

## 下载

从 [官方下载页](https://www.raspberrypi.org/downloads/raspbian/) 下载 Raspbian Stretch Lite.

## 安装

有两种可选的安装方式，可参考：[Installing operating system images](https://www.raspberrypi.org/documentation/installation/installing-images/README.md).

**方法一**

解压上一步下载的 Zip 包，将 .img 文件拷贝到 [MicroSD](https://simple.wikipedia.org/wiki/MicroSD) 卡。将 MicroSD 卡插入树莓派，然后启动电源，在系统引导下完成安装。

**方法二**

下载并安装 [Etcher](https://etcher.io/), 将上一步下载的 Zip 包或者解压后的 .img 文件通过 Etcher 工具直接刷入 MicroSD 卡。使用这种方法，可直接启动操作系统，无需在树莓派上再引导安装程序。推荐使用方法二。

> 安装好操作系统后，可以通过命令行登录系统。
> 用户名：pi
> 初始密码：raspberry
> 可通过 `passwd` 命令修改初始密码。

# 配置环境

## 配置键盘

在命令行敲命令，发现有些按键打出来的字并非键面所示，比如 `| @`，开始我还以为是键盘坏了，后来一搜才明白，这板子产自英国，默认配置为英国键盘。Raspbian 提供了配置工具，通过 `sudo raspi-config` 命令，可对一系列默认配置进行修改。

![Raspi-Config](https://i.imgur.com/kgRZ7eH.png)

通过 `4 Localisation Options` 完成对时区、地区、键盘布局等配置的设定。

## 配置 Wi-Fi

继续通过 Raspi-Config 工具的 `2 Network Options` 完成对无线网络 Wi-Fi 的配置。除此之外，也可通过手动方式完成配置，可配置多个热点，详见 [Setting WiFi up via the command line](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md).

> 至此，每次启动系统后，都会自动连接 Wi-Fi.

## 配置 SSH Server

为了方便管理树莓派，不用再连接显示器和键盘，可以开启 SSH Server，通过电脑远程登录进行管理。

依然是通过 Raspi-Config 工具的 `5 Interfacing Options` 开启 SSH。
也可以通过 `systemctl` 开启 SSH 并设置开机启动。

```
sudo systemctl enable ssh
sudo systemctl start ssh
```

详见：[Remote access - SSH (Secure Shell)](https://www.raspberrypi.org/documentation/remote-access/ssh/)

> 至此，我们可以在个人电脑上通过 SSH Client 远程登录树莓派了。
> ssh pi@ip
> ssh pi@raspberrypi.local
> 这里的 raspberrypi 是主机名，可通过 `hostname xxx` 修改主机名。

# 电源控制

配套用上小米的智能插座，可以实现远程控制树莓派的硬启动。部分智能插座型号还带 USB 接口，连电源适配器都省了。

# 结束

再想想继续在上面折腾一些什么玩意吧，把这块 100 多块钱的板子真正利用起来。