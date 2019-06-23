---
layout: post
title: "Hexo 博客 Docker 部署方案"
date: 2019-06-10 23:30:00
categories: 折腾
tags: [Docker, Hexo]
---

近期“墙哥”开启了疯狂屠杀模式，境外 VPS 横尸遍野，哥放在搬瓦工上的个人博客也不幸遇难。可耻的搬瓦工关闭了免费换 IP 服务，换一次居然要 $8.79【黑人问号】。只得另寻栖身之所，准备把博客搬到 GCP 香港。
博客系统牵涉的子系统还挺多，
* Hexo - 博客静态生成器，也可用作 Web Server 
* Nginx - 反向代理器
* Letsencrypt - 免费 SSL 证书颁发
* Webhook - 自动部署

想到以后还可能面临多次迁移，干脆趁这次机会实现容器化部署，一劳永逸了。

<!-- more -->

# 部署架构
![Deploy Arch](https://i.imgur.com/KwljC6G.png)

# 部署过程

## 安装 Docker

见网方网站：
* [Docker 安装指南](https://docs.docker.com/engine/installation/)
* [Docker-Compose 安装指南](https://docs.docker.com/compose/install/)

## Nginx-Proxy
Nginx-Proxy 可自动反向代理 Host 上启动的 Server 容器，为他们生成配置文件，重启 Nginx。
详细介绍见 [Nginx-Proxy Github Repo](https://github.com/jwilder/nginx-proxy)
```
docker run --detach \
    --name nginx-proxy \
    --publish 80:80 \
    --publish 443:443 \
    --volume /etc/nginx/certs \
    --volume /etc/nginx/vhost.d \
    --volume /usr/share/nginx/html \
    --volume /var/run/docker.sock:/tmp/docker.sock:ro \
    jwilder/nginx-proxy
```

## Letsencrypt-nginx-proxy-companion
Letsencrypt-nginx-proxy-companion 配合 Nginx-Proxy 使用，可自动为网站颁发和刷新 SSL 证书。
详细介绍见 [Github Repo](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)
```
docker run --detach \
    --name nginx-proxy-letsencrypt \
    --volumes-from nginx-proxy \
    --volume /var/run/docker.sock:/var/run/docker.sock:ro \
    jrcs/letsencrypt-nginx-proxy-companion
```

## 基于 Hexo 的博客服务
我基于 Node.js 镜像做了一个拉取代码，并自动构建和部署的镜像。
代码放在 Github 上了。见 [https://github.com/yangzhyo/iyang-docker](https://github.com/yangzhyo/iyang-docker)
```
docker run --detach \
    --name iyang-server \
    --volume /opt:/app \
    --env "VIRTUAL_HOST=iya.ng" \
    --env "LETSENCRYPT_HOST=iya.ng" \
    --env "LETSENCRYPT_EMAIL=xxxx@gmail.com" \
    yangzhyo/iyang
```

## Webhook
同样，基于 [Webhook](https://github.com/adnanh/webhook) 制作了运行 Webhook Server 的镜像，代码在 [https://github.com/yangzhyo/iyang-docker](https://github.com/yangzhyo/iyang-docker)
```
docker run --detach \
    --name iyang-webhook \
    --volume /opt:/app \
    --env "VIRTUAL_HOST=webhook.iya.ng" \
    --env "LETSENCRYPT_HOST=webhook.iya.ng" \
    --env "LETSENCRYPT_EMAIL=xxxx@gmail.com" \
    yangzhyo/iyang-webhook
```

## 大功告成

到这一步，博客已经可以以 HTTPS 的协议正常访问了，并且能支持自动更新。

## Docker-Compose

一步步的启动容器还是略显麻烦，可以使用 Docker 提供的 [Compose](https://docs.docker.com/compose/) 工具完成一键部署。
编写 `docker-compose.yml` 文件
```
version: "3"
services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - conf:/etc/nginx/conf.d
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - dhparam:/etc/nginx/dhparam
      - certs:/etc/nginx/certs:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
  letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: letsencrypt
    depends_on:
      - nginx-proxy
    volumes:
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - dhparam:/etc/nginx/dhparam:ro
      - certs:/etc/nginx/certs
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - NGINX_PROXY_CONTAINER=nginx-proxy
  iyang-server:
    image: yangzhyo/iyang
    container_name: iyang-server
    depends_on:
      - letsencrypt
    environment:
      - VIRTUAL_HOST=iya.ng
      - LETSENCRYPT_HOST=iya.ng
      - LETSENCRYPT_EMAIL=xxxx@gmail.com
    volumes:
      - /opt:/app
  iyang-webhook:
    image: yangzhyo/iyang-webhook
    container_name: iyang-webhook
    depends_on:
      - iyang-server
    environment:
      - VIRTUAL_HOST=webhook.iya.ng
      - LETSENCRYPT_HOST=webhook.iya.ng
      - LETSENCRYPT_EMAIL=xxxx@gmail.com
    volumes:
      - /opt:/app
volumes:
  conf:
  vhost:
  html:
  dhparam:
  certs:
```

运行
```
docker-compose up -d
```

停止
```
docker-compose stop
docker-compose down --volumes
```

## Swarm

如果想使用集群来运行容器，并且实现容器的自动管理，可使用 [Docker Swarm](https://docs.docker.com/engine/swarm/)

需要对 `docker-compose.yml` 文件进行适当修改，主要是给 nginx-proxy 容器贴上标签 `com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy`，这样才能使 letsencrypt-nginx-proxy-companion 正确的发现 nginx-proxy.

```
version: "3"
services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    labels:
      - com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - conf:/etc/nginx/conf.d
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - dhparam:/etc/nginx/dhparam
      - certs:/etc/nginx/certs:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
  letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    depends_on:
      - nginx-proxy
    volumes:
      - vhost:/etc/nginx/vhost.d
      - html:/usr/share/nginx/html
      - dhparam:/etc/nginx/dhparam:ro
      - certs:/etc/nginx/certs
      - /var/run/docker.sock:/var/run/docker.sock:ro
  iyang-server:
    image: yangzhyo/iyang
    depends_on:
      - letsencrypt
    environment:
      - VIRTUAL_HOST=iya.ng
      - LETSENCRYPT_HOST=iya.ng
      - LETSENCRYPT_EMAIL=xxxx@gmail.com
    volumes:
      - /opt:/app
  iyang-webhook:
    image: yangzhyo/iyang-webhook
    depends_on:
      - iyang-server
    environment:
      - VIRTUAL_HOST=webhook.iya.ng
      - LETSENCRYPT_HOST=webhook.iya.ng
      - LETSENCRYPT_EMAIL=xxxx@gmail.com
    volumes:
      - /opt:/app
volumes:
  conf:
  vhost:
  html:
  dhparam:
  certs:
```

运行
```
docker swarm init
docker stack deploy -c docker-compose.yml iyang
```

停止
```
docker stack rm iyang
docker swarm leave --force
```

# 小结
至此，博客以后再发生迁移，便可在几分钟内完成了。
