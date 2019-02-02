---
layout: post
title: "基于 SSH 反向隧道实现内网树莓派的外部访问"
date: 2019-02-01 11:28:00
categories: 折腾
tags: [SSH, Raspberry Pi, Raspbian]
---

前几天 [重装了树莓派](/2019/01/21/raspberry-pi-second-try/)，为了方便进一步把玩，需要
1. 从外部网络远程登录 Pi
2. 从外部访问 Pi 提供的 API

而 Pi 是架设在宽带网络的内网，对外没有静态 IP。为了解决这个问题，需要用到 SSH 反向隧道。

<!-- more -->

# SSH 简介

[SSH](https://www.ssh.com/ssh/protocol/) 即 [Secure Shell](https://en.wikipedia.org/wiki/Secure_Shell)，是一种安全的远程登录、命令执行和数据传输解决方案，能有效地防范中间人窃听。他用来替代 [Telnet](https://en.wikipedia.org/wiki/Telnet)，以及不安全的远程 Shell 协议，例如 Berkeley [rlogin](https://en.wikipedia.org/wiki/Rlogin)、[rsh](https://en.wikipedia.org/wiki/Remote_Shell)、[rexec](https://en.wikipedia.org/wiki/Remote_Process_Execution) 等。

## SSH 登录

### 基本用法

```
ssh -p 2222 user@host
```

* -p 指定服务器登录端口，缺省是 22
* user 指定登录用户名，若与本地用户名一致，可缺省

登录过程的基本原理是利用 [非对称加密](https://zh.wikipedia.org/zh/%E5%85%AC%E5%BC%80%E5%AF%86%E9%92%A5%E5%8A%A0%E5%AF%86)，连接时服务器将其公钥告知客户端，客户端用公钥对传输的数据进行加密，服务器用私钥解密后验证和执行。

**如何防止中间人攻击？**
1. 在首次建立连接时，客户端下载服务端的公钥，并向用户确认公钥指纹信息。确认后，将公钥信息保存到 `~/.ssh/known_hosts`

    ```
    $ ssh user@host
    The authenticity of host 'host (xxx.xxx.xxx.xxx)' can't be established.
    RSA key fingerprint is xxxxxxxxxx.
    Are you sure you want to continue connecting (yes/no)?
    ```

2. 后续建立连接时，会比较服务端公钥与首次连接时本地保存的公钥是否相同，相同才允许建立连接。

### 免密登录

客户端生成一对密钥，公钥放服务端的 `~/.ssh/authorized_keys` 文件。建立连接时，服务端先生成随机数，用公钥进行加密，发送至客户端。客户端收到后，用私钥解密，与 Session Key 一起 Hash，然后生成数字摘要发送至服务端。服务端通过同样的算法生成数字摘要与之对比，对比成功则认证通过。

## SSH 隧道

SSH 支持端口数据转发，基于这个功能可在主机之间建立通信隧道。数据传输是通过对称加密算法进行加密，算法和密钥在连接阶段进行协商。

### 本地端口转发 (Local Port Forwardings)

客户端监听一个指定端口，将该端口收到的数据直接通过 SSH 通道转发到服务端的指定端口上。可理解为正向的隧道。
```
-L [bind_address:]port:host:hostport
-L [bind_address:]port:remote_socket
-L local_socket:host:hostport
-L local_socket:remote_socket
```

### 远程端口转发 (Remote Port Forwardings)

服务端监听一个指定端口，将该端口收到的数据直接通过 SSH 通道转发到客户端的指定端口上。可理解为反向的隧道。

```
-R [bind_address:]port:host:hostport
-R [bind_address:]port:local_socket
-R remote_socket:host:hostport
-R remote_socket:local_socket
-R [bind_address:]port
```

### 从外网登录树莓派

由于树莓派在内网，且没有固定的 IP 地址，所以我们不能直接通过 SSH 登录。但可通过反向隧道来间接实现。

1. 准备一台有固定 IP 的 VPS。
2. 树莓派主动通过 SSH 建立到 VPS 的反向隧道，VPS 转发外网数据到树莓派的 SSH Server 端口。
    ```
    ssh -NfR :12346:127.0.0.1:22 -p 12345 vps_user@vps_host
    ```
    假设 vps 的 SSH 登录端口是 12345；
    假设 vps 用于转发数据的端口是 12346；
    假设数莓派的 SSH 端口是默认的 22；
    -f: Requests ssh to go to background just before command execution.
    -N: Do not execute a remote command.  This is useful for just forwarding ports.

    **一个非常重要的配置!**
    需要将 VPS 上的 SSH Server 配置文件 (/etc/ssh/sshd_config) 设置 `GatewayPorts yes`，否则转发端口的 bind address 只能是 127.0.0.1，即使在 ssh 命令中显示指定 bind address 也不会生效，会导致外网远程连接时遭到拒绝 (Connection refused)。
    ```
    $ netstat -anp | grep 12346
    tcp        0      0 0.0.0.0:12346           0.0.0.0:*               LISTEN      23458/sshd: root    
    tcp6       0      0 :::12346                :::*                    LISTEN      23458/sshd: root
    ```

3. 外网机器通过 VPS 的转发端口建立 SSH 登录。
    ```
    ssh -p 12346 pi_user@vps_host
    ```
    注意这里，端口是 VPS 的数据转发端口，vps_host 是 VPS 的地址，pi_user 是树莓派的用户名。

### 从外网连接树莓派提供的 API

与登录的原理一样，外网 -> VPS Nginx 80 端口 -> VPS 某端口 -> SSH 反向隧道 -> 树莓派的 Nginx 80 端口。
过程就不再赘述。

# AutoSSH

由于各种异常原因，如网络抖动、进程崩溃等，SSH 隧道随时可能会挂掉。
两步解决。

## SSH 配置防踢策略
树莓派的 SSH 客户端增加如下配置。
`~/.ssh/config`
```
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

## AutoSSH 失效重启
[Autossh](http://www.harding.motd.ca/autossh/) 是一个用于启动和监控 SSH 的工具, 在 SSH 进程和会话发生崩溃或异常时重启它。

**安装**
`sudo apt-get install autossh`

**运行**
```
autossh -M <port>[:echo_port] [-f] [SSH OPTIONS]
autossh -M 0 -fNR :12346:127.0.0.1:22 -p12345 vps_user@vps_host
```
值得注意的是 -f 参数为 autossh 所用，表示后台运行，不会传递给 ssh. 其他 ssh 参数则会正常传递。

**开机启动**
配置到 [Systemd](https://en.wikipedia.org/wiki/Systemd) 管理起来，增加配置文件 `/etc/systemd/system/autossh-ssh-tunnel.service`
```
[Unit]
Description=AutoSSH tunnel service - SSH
After=network.target

[Service]
Environment="AUTOSSH_GATETIME=0"
ExecStart=/usr/bin/autossh -M 0 -NR 12346:127.0.0.1:22 -p12345 vps_user@vps_host
User=pi

[Install]
WantedBy=multi-user.target
```
**特别注意**
* Environment="AUTOSSH_GATETIME=0"，说明见 [官方文档](https://www.harding.motd.ca/autossh/README.txt)。
* User=pi，设置运行 autossh 进程的用户，才能用上当前用户的 SSH 配置，如免密登录、防踢等。
* Systemd 运行不支持 autossh 的 -f 参数！

**运行以及设置开机启动**
```
systemctl daemon-reload
systemctl start autossh-ssh-tunnel.service
systemctl enable autossh-ssh-tunnel.service
```

**参考**
* [https://www.everythingcli.org/ssh-tunnelling-for-fun-and-profit-autossh/](https://www.everythingcli.org/ssh-tunnelling-for-fun-and-profit-autossh/)
* [https://www.harding.motd.ca/autossh/README.txt](https://www.harding.motd.ca/autossh/README.txt)