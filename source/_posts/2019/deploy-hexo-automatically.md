---
layout: post
title: "博客自动部署方案"
date: 2019-02-14 20:20:00
categories: 折腾
tags: [Hexo, Webhook, CI/CD]
---

前段时间把 [博客从 Jekyll 改为了 Hexo 实现](2018/04/17/blog-rebuild-with-hexo/)，站点也从 Github Pages 迁移到了 VPS，丧失了自动构建与部署的能力，好在自己折腾一下也不算麻烦。

<!-- more -->

# Hexo Deploy

Hexo 自带一个简单易用的部署工具，支持多种传输协议，详见 [官方文档](https://hexo.io/docs/deployment.html)。
支持的协议：
* Git
* Heroku
* Netlify
* Rsync
* OpenShift
* FTPSync
* SFTP

通过执行命令 `hexo d` 完成部署。

## Git

该方案会将 Hexo 构建出来的静态文件夹 `public` 推送到配置好的 Git Repo 中，例如 Github Pages Repo。如果正在使用 Github Pages 作为博客的宿主，这个方案就很合适。

在 Hexo 配置文件 `_config.yml` 修改配置：
```
deploy:
  type: git
  repo: git@github.com:yangzhyo/yangzhyo.github.io.git
```

## Rsync

该方案使用 [Rsync](https://zh.wikipedia.org/wiki/Rsync) 工具同步静态文件到远端服务器。这个方案适合自有服务器的场景。

在 Hexo 配置文件 `_config.yml` 修改配置：
```
deploy:
  type: rsync
  host: iya.ng
  user: <user>
  root: /webhost/
  port: 22
```

使用 Hexo 自带的部署工具还是不够完美，一是每次需要多执行一次部署命令，二是有可能泄漏服务器信息，如用户名、端口等。

# Git Webhook

几乎所有的 Git 服务都支持 Webhook，当 Repo 发生某些事件时（如 Push），通过 HTTP Post 调用事先配置的 Webhook URL，将事件相关的信息（Payload）通知到服务器，服务器收到通知后即可完成部署（拉取代码、执行构建、修改配置、重启服务等）。

我们不用自己写 Webhook 处理程序，因为 Github 上有成熟的工具，不必重复造轮子。推荐使用 [Adnanh Webhook](https://github.com/adnanh/webhook)，由 Go 语言编写，提供 [二进制发行版](https://github.com/adnanh/webhook/releases)，可在服务器上直接运行，无需配置运行时环境，工具也简单易用。

## Webhook Handler

1. 从 Github 上下载对应的 [二进制发行版](https://github.com/adnanh/webhook/releases)，并解压到合适的目录；
2. 编写 Webhook Handler 配置文件；
    hooks.json
    ```
    [
        {
            "id": "deploy-iyang-webhook",
            "execute-command": "/opt/webhook/deploy-iyang.sh",
            "command-working-directory": "/opt/webhook/",
            "pass-arguments-to-command":
            [
            {
                "source": "payload",
                "name": "head_commit.id"
            },
            {
                "source": "payload",
                "name": "pusher.name"
            },
            {
                "source": "payload",
                "name": "pusher.email"
            }
            ],
            "trigger-rule":
            {
                "and":
                [
                    {
                    "match":
                    {
                        "type": "payload-hash-sha1",
                        "secret": "xxxxxx",
                        "parameter":
                        {
                        "source": "header",
                        "name": "X-Hub-Signature"
                        }
                    }
                    },
                    {
                    "match":
                    {
                        "type": "value",
                        "value": "refs/heads/master",
                        "parameter":
                        {
                        "source": "payload",
                        "name": "ref"
                        }
                    }
                    }
                ]
            }
        }
    ]
    ```
    这里的字段定义见 [官方文档](https://github.com/adnanh/webhook/tree/master/docs)，从字面基本就能理解，浅显易懂。
    id：标识一个 Webhook handler，也会作为对外提供服务的 URL 地址的一部分；
    execute-command：
3. 编写部署 Shell 脚本；
    deploy-iyang.sh
    ```
    #!/bin/sh

    cd /opt/iyang
    git pull origin master
    cd /opt/iyang/themes/next
    git pull origin master
    cd /opt/iyang
    ./node_modules/hexo/bin/hexo g
    ```
4. 运行 Webhook Handler 工具。
    `nohup ./webhook -hooks hooks.json -verbose > log.txt &`
    此时，Handler 将对外提供 HTTP 服务，地址为 http://server:9000/hooks/{id}，其中 id 为 hooks.json 中配置的 id.

## 为 Git Repo 配置 Webhook

以 Github 为例，在 Repo 的 Setting 界面，找到 Webhook 菜单，即可添加 Webhook，需要
1. 配置 Payload URL（即上一节 Webhook Handler 提供的地址）；
2. Content-Type 设置为 application/json；
3. 设置 Secret，用于对 Payload 数据进行签名加盐，必须与 Webhook Handler 配置文件中的一致；
4. 选择触发事件，选择 Push 即可；

配置完成后，会自动发送一个 Ping 事件，用于测试。随后该 Repo 所有的 Push 都会触发一次 Webhook。