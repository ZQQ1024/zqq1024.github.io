---
title: "https抓包还原明文"
weight: 1
bookToc: true
---

## http和https的关系

关于HTTPS的几点理解：

- **HTTP** 本身是明文的应用层协议
- **SSL/TLS** 是一个位于 **应用层和传输层之间的加密层**，负责握手、完整性、加密、认证
- **HTTPS = HTTP over SSL/TLS**，即在 TCP 套接字上先建立 TLS 会话，再在加密的隧道里跑普通的 HTTP 报文
- 类似的还有：
  - SMTP over TLS (SMTPS)
  - IMAP over TLS (IMAPS)
  - POP3 over TLS (POP3S)
  - FTP over TLS (FTPS)
- 这样分层设计/外挂的好处是，应用层逻辑不用改感知不到SSL/TLS，安全性需求交给 TLS，即可变成安全协议

![](/data/image/network/https/image-2.png)

## SSL/TLS 握手流程

TLS1.3 复杂安全，这里处于教学目的用TLS 1.2 演示

`curl --http1.1 --tls-max 1.2 https://www.example.com`

SSL握手流程如下（单向认证、客户端认证服务器端）：

![](/data/image/network/https/image-6.png)

- Client Hello
- Server Hello, Certificate, Server Key Exchange, Server Hello Done
- Client Key Exchange, Change Cipher Spec, Encrypted Handshake Message
- Application Data

### Client Hello

客户端发起连接，携带：
- 支持的 TLS 版本
- 随机数 `Client Random`
- 支持的加密套件列表 `Cipher Suites`
- 支持的压缩方法

![](/data/image/network/https/client-hello.png)

### Server Hello, Certificate, Server Key Exchange, Server Hello Done

#### Server Hello

确认 TLS 版本、**选定**加密套件(`Cipher Suite: TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 (0xcca9)`)、`Server Random`

![](/data/image/network/https/server-hello.png)

#### Certificate

发送服务端证书（含公钥）

![](/data/image/network/https/certificate.png)

#### Server Key Exchange

发送 DH/ECDH 参数，用私钥签名，客户端用证书公钥验签，以确认没被篡改，避免中间人攻击

`Server Key Exchange` 阶段存在，说明密钥交换算法是 `ECDHE` 或 `DHE`，后续会计算出一个`Pre-Master Secret`，具备前向安全性`PFS(Perfect Forward Secrecy)`

![](/data/image/network/https/server-key-exchange.png)

{{< hint info >}}
前向安全性PFS：即使服务器的长期私钥泄露，攻击者也无法解密过去的通信记录。因为每次会话都是生成临时密钥，存于内存中的，无法还原

RSA模式下，没有`Server Key Exchange` 阶段，由客户端直接生成生成`Pre-Master Secret`，用服务器证书公钥加密，所以RSA密钥泄露能够解密出`Pre-Master Secret`
{{< /hint >}}

#### Server Hello Done

通知客户端服务端握手信息发送完毕

![](/data/image/network/https/server-hello-done.png)

### Client Key Exchange, Change Cipher Spec, Encrypted Handshake Message

#### Client Key Exchange

发送客户端 DH/ECDH 公钥（服务端不需要确认次参数是否被篡改，如果篡改会导致客户端和服务端算出的 Pre-Master Secret 不同，`Encrypted Handshake`失败）**至此双方各自可以计算出一个 `Pre-Master Secret`**，`Pre-Master Secret`从未在网络上出现过

![](/data/image/network/https/client-key-exchange.png)

{{< hint info >}}

至此，可以计算出密钥协商中的使用的各类密钥了，参看下面的**密钥协商过程**章节

{{< /hint >}}

#### Change Cipher Spec + Encrypted Handshake Message

`Change Cipher Spec` 一个通知信号，意思是："我之后发的消息都会用协商好的密钥加密了"

`Encrypted Handshake Message` 即 `Finished`，用协商好的密钥加密，验证握手完整性

![](/data/image/network/https/change-cipher-1.png)

这2个阶段，服务器端也会发送给客户端

![](/data/image/network/https/change-cipher-2.png)

### Application Data

握手完成，开始传输加密的业务数据。

![](/data/image/network/https/application-data.png)

### 总结

  |──── Client Hello ────────────→|  # client random, Cipher Suites  
  |←─── Server Hello ─────────────|  # server random, 选定 Cipher Suite  
  |←─── Certificate ──────────────|  
  |←─── Server Key Exchange ──────|  # 服务端 ecdh 参数（用私钥签名）  
  |←─── Server Hello Done ────────|  
  |──── Client Key Exchange ──────→| # 客户端 ecdh 临时公钥，至此双方各自算出对称密钥  
  |──── Change Cipher Spec ───────→| # 通知：我开始加密了  
  |──── Encrypted Handshake ──────→|  
  |←─── Change Cipher Spec ───────|  # 通知：我也开始加密了  
  |←─── Encrypted Handshake ──────|  
  |══════ Application Data ═══════|  

## 密钥协商过程

由 `Client Random` / `Server Random` / `Pre-Master Secret` ，通过`PRF`（Pseudo-Random Function，伪随机函数，结果完全确定但输出分布和真随机无法区分），可以算出 `Master Key` / `Key Material`

TLS 1.2 中 PRF 基于 `HMAC-SHA256`，核心是 `P_hash` 扩展，通过调整调用轮数确定输出长度（一轮输出32字节）：
```
A(0) = label + seed
A(1) = HMAC(secret, A(0))
A(2) = HMAC(secret, A(1))
A(3) = HMAC(secret, A(2))
...

P_hash = HMAC(secret, A(1) + seed)
       + HMAC(secret, A(2) + seed)
       + HMAC(secret, A(3) + seed)
       + ...         ↑ 拼接到够长为止
```

### `Pre-Master Secret` -> `Master Secret`

```
Master Secret = PRF(
    Pre-Master Secret,      // 密钥
    "master secret",        // 固定标签
    Client Random + Server Random  // 种子
)
```

输出固定 48 字节，两个 Random ，保证每次会话的 `Master Secret` 不同

### `Master Secret` -> `Key Material`

然后由`Master Secret`计算出协商出来的`Cipher Suite`使用的各类密钥，这里以`TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 (0xcca9)`举例

```
TLS
 └─ ECDHE          ← 密钥交换算法（临时椭圆曲线DH，有PFS）
 └─ ECDSA          ← 身份验证算法（椭圆曲线数字签名）
 └─ CHACHA20       ← 对称加密算法
 └─ POLY1305       ← MAC认证算法（和CHACHA20配套）
 └─ SHA256         ← PRF所用Hash算法

Key Material = PRF(
    Master Secret,          // 密钥
    "key expansion",        // 固定标签
    Server Random + Client Random  // 种子（注意顺序反过来了）
)
```

CHACHA20-POLY1305 是认证加密（AEAD），不需要单独 MAC key

```
<!-- 两个方向用不同的密钥，隔离两个方向的流量 -->
client write key  = 32字节  (CHACHA20 固定256位)
server write key  = 32字节  (CHACHA20 固定256位)
client write IV   = 12字节  (POLY1305 nonce)
server write IV   = 12字节  (POLY1305 nonce)
─────────────────────────────
总共需要            88字节


⌈88 / 32⌉ = 3轮
第1轮 → 32字节  (累计 32)
第2轮 → 32字节  (累计 64)
第3轮 → 32字节  (累计 96)
截断到 88字节，丢弃末尾 8字节
```

## https抓包与还原

curl 编译层面包含了 OpenSSl，curl 的 SSL/TLS 功能由 OpenSSL 实现，OpenSSL 1.1.1+ 支持 `SSLKEYLOGFILE` 环境变量，设置这个环境变量就能把握手时派生出来的密钥写入这个文件里。

`NSS(Firefox)`/`Chromium`/`OpenSSL` 等都支持导出Key Log文件格式 Key Log Format（NSS Key Log Format），Wireshark 等工具可直接解析，用来解密 SSL/TLS 流量

![](/data/image/network/https/image-5.png)

---

TLS1.3 复杂安全，这里处于教学目的用TLS 1.2 演示

`curl --http1.1 --tls-max 1.2 https://www.example.com`

keylog 文件格式是：`CLIENT_RANDOM <ClientHello Random> <Master Secret>`

![](/data/image/network/https/image-3.png)

SSLKEYLOG文件内容如下：

![](/data/image/network/https/image-4.png)

---

Wireshark 配置 keylog 文件：

- 打开 Wireshark- 菜单 Edit → Preferences（编辑 → 首选项）
- 在左侧选择 Protocols → TLS
- 找到 (Pre)-Master-Secret log filename
- 选择你的 keys.log 文件路径


![](/data/image/network/https/image-7.png)

![](/data/image/network/https/image-8.png)