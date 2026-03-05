---
title: "http"
weight: 1
bookToc: true
---

{{< markmap >}}

# HTTP

- [HTTP发展](#http-evolution)
  - WWW
  - 版本历史
- [HTTP报文结构](#http-message)
- [HTTP链接](#http-connection)
  - Keep Alive
  - Head-of-line blocking
  - Proxy
  - Upgrade
  - SSE
- [HTTP请求方法](#http-methods)
  - GET vs POST
  - 幂等性、安全性
  - 8 种方法

{{< /markmap >}}

## HTTP发展 {#http-evolution}

### WWW

[Tim Berners-Lee](https://baike.baidu.com/item/%E8%92%82%E5%A7%86%C2%B7%E4%BC%AF%E7%BA%B3%E6%96%AF%C2%B7%E6%9D%8E/8868412)，万维网(World Wide Web)的发明者，2016年获[图灵奖](https://baike.baidu.com/tashuo/browse/content?id=ea61c9a4c9282dff38ef4824)。1990年定义HTTP、HTML、URL三项关键技术，同年12月25日完成首个Web服务器与浏览器的通信。1991年创建世界上第一个网站 [info.cern.ch](https://line-mode.cern.ch/www/hypertext/WWW/TheProject.html)，1994年创立W3C推动技术标准化。

![](/data/image/network/http/www.png)

互联网(Internet)作为网络基础设施，早在 1960-70 年代就以 ARPANET 的形式存在了，主要用于军事和学术机构之间传输数据。到 1980 年代已经有相当规模，但用起来需要专业知识，普通人完全摸不着头脑。WWW技术给互联网赋予了强大的生命力，Web浏览的方式给了互联网靓丽的青春。

{{< hint info >}}
HTML = Hypertext Markup Language  
HTTP = Hypertext Transfer Protocol  

其中 Hypertext 的含义：  
文本里嵌入了可跳转的链接，点击某个词或句子可以直接跳到另一个文档或位置，而不是像书一样只能线性地从头读到尾。"超"（hyper）的意思就是超越线性，打破了传统文本只能顺序阅读的限制。这个概念最早来自 Ted Nelson。

---

万维网中的关键技术：
- HTML: 用于表示超文本的文本格式
- HTTP: 用于交换这些文本的协议
- 浏览器：用于显示（和编辑）这些文档的客户端，第一个浏览器，名为WorldWideWeb
- 服务器：用于提供文档访问的服务器，是 httpd 的早期版本
{{< /hint >}}

### 版本历史

- HTTP/0.9
  - 初始版本，当时无版本号，0.9是后续为了区分加的
  - 一行协议，只支持GET方法，`GET /my-page.html`，响应只有一个html文件`<html>xxx</html>`
  - 没有HTTP headers，错误需包含在响应中的html中
- HTTP/1.0 [RFC 1945](https://datatracker.ietf.org/doc/html/rfc1945)，先有事实标准，RFC更像是整理总结
  - 请求和响应中加入了HTTP headers，版本信息
  - 响应中加入了响应码
  - 只支持GET、HEAD、POST方法
  - 通过`Content-Type`头支持传输其他类型文档
- HTTP/1.1 RFC 1945发布之后的几个月发布了[RFC 2068](https://datatracker.ietf.org/doc/html/rfc2068)，想通过设计解决协议中的问题，后又被不断修订成多个部分
  - 链接复用 Keep-Alive、缓存机制、内容协商
  - Pipelining 机制，不用接收第一个请求的完整响应，就发送第二个请求
  - 响应支持Chunked
  - Host头，同一IP支持多个托管多个域名
  - 新增PUT、DEELTE、OPTIONS、TRACE等方法
- HTTP/2 [RFC 9113](https://datatracker.ietf.org/doc/html/rfc9113) 1.1经过了很多年的发展
  - 对原有1.1进行了压缩，性能更好，但是人读不懂
  - multiplexed 多路复用，同一个链接通过二进制分帧 + 流可以交错发送多个请求
  - 较HTTP 1.1 语义不变，只改传输机制
- HTTP/3 [RFC 9114](https://datatracker.ietf.org/doc/html/rfc9114)
  - HTTP over QUIC，解决TCP层包的重传可能block所有数据流的问题，QUIC基于UDP传输多个流，自己实现了可靠传输，互不干扰
  - 较HTTP 1.1 语义不变，只改传输机制

## HTTP报文结构 {#http-message}

![](/data/image/network/http/message.png)

HTTP 消息的起始行和标头统称为请求的头部 Head，其后包含其内容的部分称为正文 Body

## HTTP链接 {#http-connection}

### Keep Alive

HTTP 初始版本 和 1.0版本 默认使用 `short-lived connection`，即每个HTTP请求都会建立一个TCP链接，而建立TCP链接又是一个很耗时的操作。HTTP 1.1版本默认使用 `persistent connection`，或 `keep-alive connection`，即多次HTTP请求复用同一个TCP链接，通过`Keep-Alive` header 指定这个TCP链接 open多长时间，很多人口中说的HTTP长链接。

这样做也有缺陷，导致了 HOL(Head-of-line blocking) 问题的产生，后续的请求需要等待前一个请求的响应；同时就算没有HTTP请求，TCP链接也要保持open，更消耗资源

### HTTP pipelining

HTTP/1.1 引入的一种优化机制，允许客户端在不等待前一个响应的情况下，连续发送多个请求，默认浏览器没有启用该机制

![](/data/image/network/http/connection.png)

不幸的是，HTTP/1.1 要求响应必须按照接收请求的顺序返回，因此如果某个请求需要很长时间才能完成，仍然可能会发生 HOL 问题。

### Head-of-line blocking

分为 `application-level` 和 `transport-layer`  HOL blocking

`application-level` HOL blocking 类似上面的HTTP 1.1 及pipelining中描述的问题，HTTP/2 通过 multiplexing 多路复用的方式解决了`application-level`的问题。

HTTP/2 在单个 TCP 连接上使用多个带编号的流，交错发送多个请求和响应，而流优先级则有助于服务器决定首先发送哪些流

![](/data/image/network/http/multiplexing.png)

传输层的丢包仍然会导致 HOL blocking，HTTP/2 运行在 TCP 之上，丢失的 TCP segment 会使该连接上所有流 blocking，直到丢失的数据被重传

HTTP/3 通过使用 QUIC over UDP 解决了`transport-layer`  HOL blocking的问题，多个独立的流互不干扰

### Proxy

分为正向代理`forward proxies`和方向代理`reverse proxies`，正向代理针对的目标是client，如向client通过代理访问互联网；反向代理针对的是内部服务，如隐藏内部服务身份。

我们可以通过 `X-Forwarded-For` 头获得访问代理前client的原始IP。

---

可以设置以下环境变量来让curl等命令通过代理访问，但为什么设置的都是 `http://127.0.0.1:30081`？
```bash
http_proxy=http://127.0.0.1:30081
https_proxy=http://127.0.0.1:30081
    ↑            ↑
这是"目标流量类型"  这是"与代理通信的协议"
```

首先这里的 `http://` 指的是客户端到代理的协议，只是说明用HTTP 协议连接这个代理

**当 curl 访问 https://example.com 时**，会向代理发一个特殊的 HTTP 请求：
```http
CONNECT example.com:443 HTTP/1.1
Host: example.com:443

HTTP/1.1 200 Connection Established
```

代理收到后链接就会进入透明模式，转发TCP流，这个叫做 `HTTP CONNECT` 隧道，代理并不感知内容，TLS 握手还是直接发生在 curl 和目标服务器之间
```
curl ──TCP_A── proxy ──TCP_B── example.com

read(TCP_A) → write(TCP_B)
read(TCP_B) → write(TCP_A)
```

---

回到HTTP，**当 curl 访问 http://example.com 时**，会向代理发包含绝对路径的 HTTP 请求
```http
GET http://example.com/index.html HTTP/1.1
Host: example.com
```

代理解析这个请求，知道目标是 `example.com`，然后自己发起第二段请求，注意变为了相对路径
```http
GET /index.html HTTP/1.1
Host: example.com
```

```
client ──HTTP── proxy ──HTTP── example.com
```

### Upgrade

WebSocket 本质是和 HTTP 同一层级但完全不同的协议，可以实现浏览器和服务器的相互通信

通过 HTTP 协议 Upgrade 来完成 WebSocket 的建立，相当于复用已有的 TCP 连接，浏览器不用暴露低级别的TCP API，能复用现有的基础设施

```http
GET /chat HTTP/1.1
Host: example.com
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```http
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

HTTP 升级完成后，传输的不再是 HTTP 报文，而是 **WebSocket 帧**，轻量的二进制帧，没有 HTTP 的 header 开销，每次通信不需要重复携带 Cookie、UA 等信息，这里不展开介绍。

`Sec-WebSocket-Key` 是用来确认服务器是支持 `WebSocket` 的，通过返回 `Sec-WebSocket-Accept` 来确认（客户端发送 WebSocket 握手请求，但服务器是一个普通 HTTP 服务器，不理解 WebSocket，服务器可能返回 200 OK，客户端误以为升级成功，开始发 WebSocket 帧导致错乱）
```
Sec-WebSocket-Accept = 
base64(SHA1(key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))
                            ↑
                  魔法字符串，写死在 RFC 6455 里
```

### SSE

SSE(Server-sent events) 是单向的服务器推送，基于普通 HTTP，不需要协议升级。

```
GET /events HTTP/1.1
Accept: text/event-stream ← 关键 header
```

```
HTTP/1.1 200 OK
Content-Type: text/event-stream  ← 关键
Cache-Control: no-cache
Connection: keep-alive

                                 ← 连接不关闭，持续发数据
```

之后服务器**保持连接不断**，不断往里写数据。ChatGPT/Calude 对话的流式输出用的就是SSE。


## HTTP请求方法 {#http-methods}

## HTTPS {#https}

## HTTP/2 {#http2}

## 跨域问题 {#cors}

## 缓存 {#cache}

## Cookie 字段 {#cookie}