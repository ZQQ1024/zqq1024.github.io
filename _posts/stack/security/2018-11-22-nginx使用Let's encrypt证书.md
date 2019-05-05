---
title: "nginx使用Let's encrypt证书"
categories:
  - security
tags:
  - certificate
classes: wide

excerpt: "nginx上配置使用Let's encrypt证书"
---

Let’s Encrypt is a `free`, `automated`, and `open` Certificate Authority.

如下说明nginx如何使用Let's encrypt证书：

1. 安装certbot
```
$ add-apt-repository ppa:certbot/certbot
$ apt update
$ sudo apt-get install python-certbot-nginx
```

2. 申请证书
```
$ certbot --server https://acme-v02.api.letsencrypt.org/directory -d \
*.chislab.com --manual --preferred-challenges dns-01 certonly
```
为`chislab.com`申请一个wildcard证书

3. 验证域名  
上述命令后，会提示类似如下信息：
```
Please deploy a DNS TXT record under the name
_acme-challenge.chislab.com with the following value:

G_UaEUAAoGQChpXRNEA6F4YNRh67t95XzLDUnglTneY

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```
通过配置一个`DNS TXT`记录来验证您是否为域名的拥有者

4. nginx配置  
域名验证通过后，会在如下类似位置生成证书和私钥：
```
/etc/letsencrypt/live/chislab.com
```
在`server`块下，添加如下配置，指明证书和私钥位置：
```
ssl_certificate /etc/letsencrypt/live/chislab.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/chislab.com/privkey.pem;
```