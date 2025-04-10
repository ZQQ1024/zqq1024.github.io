---
title: "DNS"
weight: 2
bookToc: true
---

## 1 背景

在此之前主要使用的代理软件如下：
- `V2RayN`：macOS下 V2Ray GUI 客户端，主要解决浏览器使用问题
- `Proxifier`：搭配使用，使不支持代理的工具/命令等使用代理，如终端上执行`curl www.google.com`走代理

`V2RayN` 使用了 `PAC` 模式，V2RayN 中使用的 v2ray-core 路由配置是全局配置，通过 PAC 列表控制哪些域名走这个全局代理

Proxifier上设置了相关规则，设置了 curl 访问 www.google.com 时走代理

---

由于 V2RayN 目前项目已经不再维护，同时目前市面上的 V2Ray GUI 程序都无法满足个人的一些特定需求，最终放弃了 GUI 客户端的使用，改直接使用 v2ray-core，其中使用了`ByPassMainland`和`GFWList`2种路由：

- `ByPassMainland`：即所有非`geoip:cn`的请求都走代理，配置如下：
{{< expand  "..." >}}
```json
{
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "outboundTag": "direct",
        "domain": [
          "geosite:cn",
        ],
        "enabled": true
      },
      {
        "type": "field",
        "outboundTag": "direct",
        "ip": [
          "geoip:private",
          "geoip:cn"
        ],
        "enabled": true
      },
      {
        "type": "field",
        "port": "0-65535",
        "outboundTag": "proxy",
        "enabled": true
      }
    ]
  }
}
```
{{< /expand >}}
- `GFWList`: 即所有在 `GFW` 列表中的站点走代理，配置如下：
{{< expand  "..." >}}
```json
{
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "outboundTag": "proxy",
        "domain": [
          "geosite:gfw",
          "geosite:greatfire",
          "geosite:tld-!cn",
          "geosite:speedtest"
        ],
        "enabled": true
      },
      {
        "type": "field",
        "port": "0-65535",
        "outboundTag": "direct",
        "enabled": true
      }
    ]
  }
}
```
{{< /expand >}}

## 2 问题

终端上执行 `curl www.google.com`，发现不论路由是 `ByPassMainland` 还是 `GFWList`，`curl`**都无法正确返回**

查看 V2Ray 日志，发现请求到 V2Ray 时 Proxifier 已经将域名解析成了IP，本地设置的 DNS 服务器为`8.8.8.8`，因为 **DNS 污染**的缘故所以无法正确返回

个人遂打算系统性的解决 DNS 污染这个问题，调研出了以下几种解决方案，几种方案各有取舍，需要结合需求使用

## 3 方案

### 3.1 Proxifier
可以通过 Proxifier 的`DNS - Resolve hostnames through proxy`功能实现

在调整 Proxifier 设置后，发现只有在 `ByPassMainland` 路由模式下，才能正确返回，为了解释这一现象，查阅相关资料理解了 Proxifier 通过代理解析域名的工作机制

#### 3.1.1 Fake-IP

GFW 不仅对某些特定的 IP 进行了阻断，还对相当一部分域名采取了 DNS 污染。简单来说，就是通过域名解析到的 IP 是错误的，所以我们还需要对 DNS 请求进行代理。但每次请求如果都是先请求远程服务器 DNS，再返回解析结果，客户端再请求，再代理，这样一来二去就极大浪费了时间。于是，Fake-IP 技术出现了。

所谓 Fake-IP，也就是虚假的IP。每次 DNS 请求，如果匹配了当前 Proxifer 设置的代理规则，会被Proxifer 软件劫持，并立刻返回一个`0.*.*.*`样式的`虚假IP`。于此同时，Proxifier 会向代理发送DNS 请求，通过代理向`8.8.8.8`发送查询`www.google.com`的 `A` 记录，将获取到的`真实IP`和`虚假IP`做映射保存在自己的表中，后续面向`虚假IP`的链接会被 Proxifier 对应的`真实IP`替换(DestOverride)

为什么在`ByPassMainland`路由模式下，`curl www.google.com`才能正确返回，但`GFWList`路由模式不行，因为发向`8.8.8.8`的DNS查询请求在`ByPassMainland`路由模式下会被`proxy`而不是`direct`，能够通过代理获取到`真实IP`而不是污染后的结果

#### 3.1.2 优缺点

该模式优点在于节省了等待 DNS 从代理获取`真实IP`的时间，缺点是由于返回了`虚假IP`，污染了本地DNS 缓存，在退出 Proxifer 软件后，会导致对应的网络问题

### 3.2 国外DoH/DoT

使用国外 DoT(DNS over TLS)/DoH(DNS over HTTPS)也可以解决 DNS 污染问题

#### 3.2.1 Cloudfare DoH

```bash
$ cloudflared --version
$ cloudflared proxy-dns --port 5555

$ dig +short @127.0.0.1 -p5555 google.com A
142.250.191.68
$ dig +short @127.0.0.1 -p5555 baidu.com A
www.a.shifen.com.
www.wshifen.com.
104.193.88.123
104.193.88.77
```
{{< hint info >}}
这里发现通过国外 DoH 解析到的`104.193.88.77`为百度在国外的IP
{{< /hint >}}

> 可以参看：[https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/dns-over-https-client/](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/dns-over-https-client/)

#### 3.2.2 优缺点

可以看到DNS能够正确解析了，但是缺点明显
- **无DNS分流**，所有DNS请求会跑到国外转一圈，延迟高
- **国内CDN不友好问题**，如果国内网站在国外使用了CDN服务，会绕道到国外访问

### 3.3 国内域名分流

#### 3.3.1 DNSmasq + dnsmasq-china-list

`DNSmasq`也可以用`SmartDNS`等替代

[dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list) 这类分流方案的核心是维护一个常用的国内域名列表，当查询列表内域名的IP时，DNS服务器选用国内DNS查询，其它情况下，使用国外DNS查询

这种方案下：  
DNSmasq + dnsmasq-china-list 用于处理DNS请求，V2Ray 用于处理代理

#### 3.3.2 优缺点

由于V2Ray使用的是[v2ray-rules-dat](https://github.com/Loyalsoldier/v2ray-rules-dat)或[domain-list-community](https://github.com/v2fly/domain-list-community)来判断站点和IP是否来自国内是否要直连，而DNS解析使用的是[dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list)，二者并不一致，可能会出现以下情况：
- 国内DNS解析出国内IP后走代理
- 国外DNS解析出国外IP后直连

不考虑是否能实现的情况下，解决不一致的方案思路有以下几种：
- 让 DNS 服务器使用 V2Ray 的`domain-list-community`
- 让V2Ray使用`dnsmasq-china-list`
- 让 V2Ray 和 DNS 服务器都用`v2ray-rules-dat`

下面介绍的V2Ray DNS方案就是让 V2Ray 也作为DNS服务器，以解决DNS请求和代理时不一致的问题

### 3.4 V2Ray DNS

Kitsunebi 的作者在 [《漫谈各种黑科技式 DNS 技术在代理环境中的应用》](https://medium.com/@TachyonDevel/%E6%BC%AB%E8%B0%88%E5%90%84%E7%A7%8D%E9%BB%91%E7%A7%91%E6%8A%80%E5%BC%8F-dns-%E6%8A%80%E6%9C%AF%E5%9C%A8%E4%BB%A3%E7%90%86%E7%8E%AF%E5%A2%83%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8-62c50e58cbd0)中介绍了通过 `Dokodemo` 入站协议和 `DNS` 出站协议开放 V2Ray DNS 的方法，可以让 V2Ray 作为 DNS服务器

该方法的核心思想是使用 `Dokodemo` 入站协议接收 DNS 请求流量，转发至 `DNS` 出站协议。而 DNS 出站协议会拦截`A`和`AAAA`的 DNS 查询并交由 V2Ray 内置 DNS 处理，从而返回查询结果；除此之外的查询流量，会根据 `Dokodemo` 的配置发送至目标 DNS 服务器

#### 3.4.1 配置

具体配置如下
{{< expand  "..." >}}
```json
{
  "dns": {
    "servers": [
      {
        "address": "223.5.5.5",
        "port": 53,
        "domains": [
          "geosite:cn"
        ]
      },
      {
        "address": "https+local://1.1.1.1/dns-query",
        "domains": [
          "geosite:geolocation-!cn"
        ]
      },
      "114.114.114.114"
    ]
  },
  "inbounds": [
      {
          "tag": "dns-in",
          "port": 53,
          "protocol": "dokodemo-door",
          "settings": {
              "address": "223.5.5.5",
              "port": 53,
              "network": "tcp,udp"
          }
      }
  ],
  "outbounds": [
      {
          "protocol": "dns",
          "tag": "dns-out"
      }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "inboundTag": [
            "dns-in"
        ],
        "outboundTag": "dns-out",
        "enabled": true
      }
    ]
  }
}
```
{{< /expand >}}

配置的主要含义是：
- 发往本地53端口的`A`和`AAAA`记录的DNS 请求会被拦截
- 如果待查询域名匹配 `geosite:cn`，则使用国内阿里云的`223.5.5.5`查询
- 如果待查询域名匹配 `geosite:geolocation-!cn`，即为常见的非国内站点的域名，则使用国外`https+local://1.1.1.1/dns-query`进行查询，`local`表示DNS查询请求不会被`routing`路由模块处理，由本地直接发出
- 如果都不满足上述，则使用`114.114.114.114`进行查询
- 除`A`和`AAAA`之外的查询流量，会根据`Dokodemo`的配置发送至目标`223.5.5.5`服务器

> 其他配置参看：[https://www.v2fly.org/en_US/config/dns.html#dnsobject](https://www.v2fly.org/en_US/config/dns.html#dnsobject)

#### 3.4.2 优缺点

优点是 DNS 分流，同时避免了 **国内 DNS 解析出国内 IP 后走代理/国外 DNS 解析出国外 IP 后直连** 的情况，缺点是由于 DNS 协议的复杂性，V2Ray DNS 并不是一个专门的 DNS 服务器，目前仅只支持`A`和`AAAA`记录，查询`TXT`得到的结果仍然是污染后的结果，如果只关心`A`和`AAAA`记录的准确性，则该缺点可忽略

### 3.5 CoreDNS + V2Ray DNS

用 `CoreDNS` 作为 frontend，将A和AAAA记录的请求和其他类型请求分开，V2Ray DNS作为backend专门处理A和AAAA记录的请求

#### 3.5.1 配置

可以使用`view`插件，将A和AAAA记录的请求和其他类型请求分开，`Corefile`如下：
{{< expand  "..." >}}
```
. {
  view v2ray-dns {
    expr type() in ['A', 'AAAA']
  }
  forward . 127.0.0.1:5301
  health
  cache
}

. {
  forward . tls://1.1.1.1 tls://1.0.0.1 {
    tls_servername cloudflare-dns.com
  }
  health
  cache
}
```
{{< /expand >}}

{{< hint info >}}
注意修改V2Ray inbounds监听端口为`5301`
{{< /hint >}}

#### 3.5.2 优缺点

优点是比较理想的解决了DNS污染问题，缺点是引入了新的CoreDNS组件，加长了请求链路，增加了复杂性

## 4 总结

总结一下针对 DNS 污染的解决方案：
- `Proxifier`利用了`Fake-IP`，简单，会污染本地DNS
- `国外DoH/DoT`一刀切无DNS分流，准确，慢
- `dnsmasq-china-list`国内域名分流，简单，存在偶发不一致情况
- `V2Ray DNS`能DNS分流，暂无法解决除A和AAAA记录的其他请求
- `CoreDNS + V2Ray DNS`较为理想的解决方案，复杂
