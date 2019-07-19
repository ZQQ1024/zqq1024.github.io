---
title: "(NGINX WAF 1)NGINX和ModSecurity的安装"
categories:
  - security
tags:
  - Nginx
  - WAF
classes: wide

excerpt: "NGINX和ModSecurity的安装"
---

# 1 介绍

本文只要是给NGINX配置ModSecurity，来提供基于规则的WAF防护功能(系统Ubuntu 16.04)

关于ModSecurity介绍：
1. ModSecurity
protects applications against a broad range of **Layer 7 attacks**, such as SQL
injection (SQLi), local file inclusion (LFI), and cross‐site scripting (XSS).
2. **2017**: ModSecurity 3.0 released with support for NGINX and NGINX Plus
3. Previous versions of ModSecurity, up through version 2.9, did technically
work with NGINX, but suffered from **poor performance**. This is because
ModSecurity was wrapped inside a full version of the Apache HTTP Server,
which provided a compatibility layer.
4. This new architecture in version 3.0 moves the core functionality into a
stand‐alone engine called **libmodesecurity**. The libmodesecurity
component **interfaces with NGINX** through an optimized connector called
the NGINX connector.
5. The combined package of libmodsecurity along with the NGINX connector
create the ModSecurity dynamic module for NGINX（侧重点在组合，不是动态，static module应该也包含）
6. ModSecurity 3.0 does not yet have feature parity with the previous version,
ModSecurity 2.9：
   - Rules that **inspect the response body** are not supported
   - Inclusion of the **request and response body in the audit log** is not supported

分成三部分吧：
1. NGINX和ModSecurity的安装
2. NGINX配置ModSecurity测试拦截效果
3. ~~配置Elasticsearch和Kibana作Dashboard~~（没有弄出来）


这是第一部分

# 2 安装
## 2.1 安装NGINX
NGINX 1.11.5 or later is required.

Nginx`Mainline`和`Stable`版本中选择`Stable`，不`Compiling from Source`而选择`Prebuilt Package`

1.Download the key used to sign NGINX packages and the repository, and add it to the apt program’s key ring:
```
$ sudo wget https://nginx.org/keys/nginx_signing.key
$ sudo apt-key add nginx_signing.key
```

2.Edit the **/etc/apt/sources.list** file, for example with vi:
```
deb https://nginx.org/packages/mainline/ubuntu/ <CODENAME> nginx
deb-src https://nginx.org/packages/mainline/ubuntu/ <CODENAME> nginx
```
我用的是Ubuntu 16.04，将`<CODENAME>`替换为`xenial`

3.Install NGINX Open Source（对应是商业版的NGINX Plus）:
```
$ sudo apt-get remove nginx-common
$ sudo apt-get update
$ sudo apt-get install nginx
```

4.Show NGINX version, and start NGINX Open Source:
```
$ nginx -v 
nginx version: nginx/1.15.3
$ sudo nginx
```

5.Verify that NGINX Open Source is up and running(Fetch the HTTP-header only):
```
$ curl -I 127.0.0.1
HTTP/1.1 200 OK
Server: nginx/1.15.3
xxx 略
```

## 2.2 安装ModSecurity
主要是compile ModSecurity as an NGINX dynamic module

In ModSecurity 3.0’s new modular
architecture, **libmodsecurity** is **the core component** which includes all
rules and functionality.**The second main component** in the architecture is a
**connector** that links libmodsecurity to the web server it is running with.

1.Install prerequisite packages:
```
$ apt-get install -y apt-utils autoconf automake build-essential git libcurl4-openssl-dev \
libgeoip-dev liblmdb-dev libpcre++-dev libtool libxml2-dev libyajl-dev pkgconf wget zlib1g-dev
```

2.Download and compile libmodsecurity:
```
$ git clone --depth 1 -b v3/master --single-branch \
https://github.com/SpiderLabs/ModSecurity
$ cd ModSecurity
$ git submodule init
$ git submodule update
$ ./build.sh
$ ./configure
$ make
$ make install
```
The compilation takes about 15 minutes, depending on the processing power
of your system.
出现下面信息请忽略，无关紧要：
```
fatal: No names found, cannot describe anything.
```

3.Download the NGINX Connector for ModSecurity and compile it
as a dynamic module:
```
$ git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
$ nginx -v
nginx version: nginx/1.15.3
$ wget http://nginx.org/download/nginx-1.15.3.tar.gz
$ tar zxvf nginx-1.15.3.tar.gz

$ cd nginx-1.15.3
$ ./configure --with-compat --add-dynamic-module=../ModSecurity-nginx
$ make modules
$ cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules
```
注意：Download the source code corresponding to the installed version of
NGINX (the complete sources are required even though only the dynamic
module is being compiled)

4.Load the NGINX ModSecurity Connector dynamic module(in the top‐level (“main”) context of **/etc/nginx/nginx.conf** ):
```
load_module modules/ngx_http_modsecurity_module.so;
```

5.Verify that the module loads successfully:
```
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```