---
layout: post
title: "基于 Let’s Encrypt 开启 HTTPS"
date: 2019-01-22 20:31:00
categories: 折腾
tags: [SSL,HTTPS]
---

[HTTPS](https://en.wikipedia.org/wiki/HTTPS) 目前已经是大多数成熟网站的标配，它通过在浏览器与网站之间建立加密信道来保证数据交换的隐私与完整性。

<!-- more -->

# 证书入门

HTTPS 建立过程不在这里展开讲，在网上能搜到大量相关论文，后面有机会再作分享。核心的逻辑是，通过 [非对称加密算法](https://zh.wikipedia.org/wiki/%E5%85%AC%E5%BC%80%E5%AF%86%E9%92%A5%E5%8A%A0%E5%AF%86) 交换密钥，然后通过 [对称加密](https://zh.wikipedia.org/wiki/%E5%B0%8D%E7%A8%B1%E5%AF%86%E9%91%B0%E5%8A%A0%E5%AF%86) 完成数据加密传输。为什么是这样？原因很简单，对称加密的计算效率远远高于非对称加密，是出于性能的考虑。

我们知道非对称加密有一对公钥/私钥，私钥只能自己掌握，公钥对外公开。
* 私钥加密的内容只有公钥能解开，这个特性可用于 [数字签名](https://zh.wikipedia.org/wiki/%E6%95%B8%E4%BD%8D%E7%B0%BD%E7%AB%A0)，做数据完整性验证。例如我用我的私钥对一份文件进行签名，张三用我的公钥可以对签名进行验证，如果验证成功则表示该文件没有被篡改。
* 公钥加密的内容只有私钥能解开，这个特性可用于数据加密传输。例如我要向张三传输一份绝密文件，就可以使用张三的公钥加密后进行传输，除了张三之外任何人都无法解密这份文件，即使中间窃取了内容也无可奈何。

既然公钥是公开的，那这里就存在一个难题：如何确保公钥真的就是张三的？会不会是攻击者李四欺骗大家，宣称自己是张三？

大家看过《三体》，知道猜疑链，两个参与者之间不可能达成直接的信任！这时候就引入了大家都信任的第三方，由第三方来证明公钥就是张三的。这个第三方机构就是 [数字证书认证机构(Certificate Authority，缩写为CA)](https://zh.wikipedia.org/wiki/%E8%AF%81%E4%B9%A6%E9%A2%81%E5%8F%91%E6%9C%BA%E6%9E%84)，她所颁发的证明文件就是 CA 证书。

CA 中心必须是公信力足够强的机构，不会造假，这是大家相互信任的基石。所以浏览器里面内置了很多公信力被认可的机构，只有这些机构签发的证书才可用于建立受信任的安全通道，否则浏览器会发出安全警告。同时，我们也会看到，经常有一些 CA 机构因违反了规定被浏览器移出信任列表，例如国内某知名证书认证机构就因未经认证为域名颁发证书而被吊销信任。

# SSL 证书类型

根据不同的验证级别，分为
* DV (Domain validated) Certificates，只验证域名的所有权，一般用于个人网站、小型网站。
* OV (Organization validated) Certificates，除了验证 DV 所需要的信息外，还需要验证组织相关的信息，详见 [The information required for OV certificates](https://www.ssl.com/faqs/ssl-ov-validation-requirements/)。这类证书会包含组织名称，一般用于公司、政府、其他想要提升信任级别的实体。
* EV (Extended validation) Certificates，比 OV 更高的验证级别，需要更严苛的认证信息，详见 [EV SSL Requirements](https://www.ssl.com/faqs/ssl-ev-validation-requirements/)。这类证书认证的域名，浏览器会在地址栏用绿色显示其公司名，以表达强力的信任程度，如下所示。
![EV Domain](https://cdn.ssl.com/app/uploads/2015/07/DVOVEV_all.png?x10733)

更多信息，可参考：[DV OV and EV Certificates](https://www.ssl.com/article/dv-ov-and-ev-certificates/)

# Let’s Encrypt

[Let’s Encrypt](https://letsencrypt.org/about/) 是国际上著名的免费 CA 认证中心，提供开放的、自动化的认证服务。

## 部署

如果有服务器 Shell 访问权限，可使用 [Certbot](https://certbot.eff.org/) 完成自动化部署。
如果没有服务器 Shell 访问权限，请参考 [官方指南](https://letsencrypt.org/getting-started/)。

## Certbot

[Certbot](https://certbot.eff.org/) 是由 [电子前沿基金会，The Electronic Frontier Foundation - EFF](https://www.eff.org/about) 以 [ACME 协议](https://ietf-wg-acme.github.io/acme/draft-ietf-acme-acme.html) 为基础开发的 SSL/TLS 证书自动获取和部署工具。

通过 [Certbot](https://certbot.eff.org/) 官网选择操作系统和 Web Server 软件，获取对应的部署指南。以 CentOS 7 + Nginx 为例，见 [https://certbot.eff.org/lets-encrypt/centosrhel7-nginx](https://certbot.eff.org/lets-encrypt/centosrhel7-nginx).

### 安装 Certbot

```shell
yum -y install yum-utils
yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional
sudo yum install python2-certbot-nginx
```

### 部署证书

```shell
sudo certbot --nginx
```

这个命令会自动获取证书，并修改 Nginx 配置文件。
![Certbot-Nginx](https://i.imgur.com/KP0AgIs.png)

如果你只想获取证书，然后手动配置 Nginx，可以加上 `certonly` 参数。

```shell
sudo certbot --nginx certonly
```

如果想使用通配符证书，需要用到 [Certbot's DNS plugins](https://certbot.eff.org/docs/using.html#dns-plugins)，详细用法参考 [官方指南](https://certbot.eff.org/lets-encrypt/centosrhel7-nginx)。

### 更新证书

CA 证书是有有效期的，Let’s Encrypt 的证书有效期为 3 个月，到期后需要更新，否则浏览器会对 HTTPS 进行安全警告。
通过执行以下命令，完成证书的更新。

```
sudo certbot renew --dry-run
```

如果怕忘记，可以配置一个 crontab 定时任务，自动完成检查和更新。

```
0 0,12 * * * python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew 
```