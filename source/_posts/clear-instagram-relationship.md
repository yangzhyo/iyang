---
layout: post
title: "Instagram 僵尸清理记"
date: 2018-12-31 21:22:00
categories: 折腾
tags: [Instagram]
---

> Instagram 被老毛子盗窃了，邮箱都被篡改了，好在最终找回了帐号。登录帐号发现，多了几百个粉丝（Followers），两千多个关注（Followings），更要命的是，Instagram 并没有提供批量移除的功能，总不可能人肉一个个干，万幸的是程序员不怕折腾。
> Tags: 批量移除 Instagram 内容、关注、粉丝，Remove Instagram media, followers, followings.

<!-- more -->

# Instagram API Platform

第一个能想到的办法是看看 [Instagram Open API](https://www.instagram.com/developer/)，但官宣老版的 API Platform 已经关闭了，全部迁移到 [Facebook 图谱 API](https://developers.facebook.com/docs/graph-api?locale=zh_CN) 的子项目 [Instagram Graph API](https://developers.facebook.com/products/instagram/)

![https://i.imgur.com/FTASMl6.png](https://i.imgur.com/FTASMl6.png)


# Instagram Graph API

> 图谱 API 是读取和写入 Facebook 社交关系图谱的主要途径。我们所有的 SDK 和产品都能以某种方式与图谱 API 互动，我们其他的 API 都是图谱 API 的扩展，因此了解图谱 API 的工作方式至关重要。

图谱 API 得名于“社交关系图谱”理念 — Facebook 上的一种信息表示形式。它由以下部分组成：

* 节点 — 用户、照片、主页或评论等基本的个体对象
* 连线 — 对象集合与单个对象之间的联系，如主页上的照片或照片的评论
* 字段 — 关于对象的数据，如用户的生日或主页的名称

更多关于图谱 API 的概念和架构，可参考 [https://developers.facebook.com/docs/graph-api/overview/](https://developers.facebook.com/docs/graph-api/overview/)

再来看看 Instagram Graph API 的文档 [https://developers.facebook.com/docs/instagram-api](https://developers.facebook.com/docs/instagram-api)

![https://i.imgur.com/4O2ou8B.png](https://i.imgur.com/4O2ou8B.png)

大致浏览了目前已开放的接口，并未找到 Relationship 相关的接口！用不了。


# Instagram SDK

Github 上有 Instagram 官方提供的 SDK，见：[https://github.com/facebookarchive/python-instagram](https://github.com/facebookarchive/python-instagram)

## 安装
```python
pip install python-instagram
```

## 测试
```python
from instagram.client import InstagramAPI

access_token = "XXXXXXXX"
client_secret = "XXXXXXXX"
api = InstagramAPI(access_token=access_token, client_secret=client_secret)
follows, next_ = api.user_follows()
```
如何获取 access token 请参考文档 [https://www.instagram.com/developer/authentication/](https://www.instagram.com/developer/authentication/)

## 结果
```python
Traceback (most recent call last):
  File "./in.py", line 6, in <module>
    follows, next_ = api.user_follows()
  File "/usr/lib/python2.7/site-packages/instagram/bind.py", line 197, in _call
    return method.execute()
  File "/usr/lib/python2.7/site-packages/instagram/bind.py", line 189, in execute
    content, next = self._do_api_request(url, method, body, headers)
  File "/usr/lib/python2.7/site-packages/instagram/bind.py", line 163, in _do_api_request
    raise InstagramAPIError(status_code, content_obj['meta']['error_type'], content_obj['meta']['error_message'])
instagram.bind.InstagramAPIError: (400) APINotAllowedError-This endpoint has been retired
```

这套 SDK 背后就是 Instagram API Platform，证实其确实已经关闭！不能用。


# Instagram Private API

API Platform 关闭以后，有网友通过逆向工程研究公开了 Instagram Private API, 并提供了 SDK, 列举几个 Github Repo

* [https://github.com/mgp25/Instagram-API](https://github.com/mgp25/Instagram-API)
* [https://github.com/LevPasha/Instagram-API-python](https://github.com/LevPasha/Instagram-API-python)
* [https://github.com/dilame/instagram-private-api](https://github.com/dilame/instagram-private-api)

mgp25 的 PHP 的 SDK 亲测可用！我将代码放在了 Github 上。
[https://github.com/yangzhyo/clear-instagram](https://github.com/yangzhyo/clear-instagram)

## 安装
```php
composer require mgp25/instagram-php
```

## 代码
```php
<?php
require __DIR__.'/../vendor/autoload.php';

$ig = new \InstagramAPI\Instagram();
$ig->setProxy('socks5://127.0.0.1:1086'); // 这个代理设置，你懂的
$ig->login('test','test');

// 移除关注
$followings = $ig->people->getSelfFollowing('123e4567-e89b-12d3-a456-426655440000');
print_r('Totally '.count($followings->getUsers())." followings\n");
foreach ($followings->getUsers() as $user) {
    print_r('Remove '.$user->getPk()."\n");
    $ig->people->unfollow($user->getPk());
}

// 移除粉丝
$followers = $ig->people->getSelfFollowers('123e4567-e89b-12d3-a456-426655440000');
print_r('Totally '.count($followers->getUsers())." followers\n");
foreach ($followers->getUsers() as $user) {
    print_r('Remove '.$user->getPk()."\n");
    $ig->people->removeFollower($user->getPk());
}

// 移除媒体
$userFeeds = $ig->timeline->getSelfUserFeed();
print_r('Totally '.count($userFeeds->getItems())." media\n");
foreach ($userFeeds->getItems() as $item) {
    print_r('Delete '.$item->getPk()."\n");
    $ig->media->delete($item->getPk());
}
```

## 结果
运行成功，但是要注意 Instagram 的频控，如果被中止过一会儿重试就好了。