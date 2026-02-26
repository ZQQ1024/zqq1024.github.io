---
title: "https抓包还原明文"
weight: 1
bookToc: true
---

## http和https的关系

![](/data/image/network/https/image.png)

![](/data/image/network/https/image-1.png)

- **请求方法**：GET/POST/PUT/DELETE/HEAD/OPTIONS
- **请求头/响应头**：User-Agent/Cookie/Set-Cookie/Content-Type/Access-Control-Allow-*/...
- **响应码**：200/301/302/400/403/500/503
- **请求体/响应体**：表单/文件/json/html等

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

## https抓包与还原

curl 编译层面包含了 OpenSSl，curl 的 SSL/TLS 功能由 OpenSSL 实现，OpenSSL 1.1.1+ 支持 SSLKEYLOGFILE 环境变量，设置这个环境变量就能把握手时派生出来的密钥写入这个文件里。

NSS(Firefox)/Chromium/OpenSSL 等都支持导出Key Log文件格式 Key Log Format（NSS Key Log Format），Wireshark 等工具可直接解析，用来解密 SSL/TLS 流量

![](/data/image/network/https/image-5.png)

---

TLS1.3 复杂安全，这里处于教学目的用TLS 1.2 演示

keylog 文件格式是：`CLIENT_RANDOM <ClientHello Random> <Master Secret>`

![](/data/image/network/https/image-3.png)

SSLKEYLOG文件内容如下：

![](/data/image/network/https/image-4.png)


SSL握手流程如下（每个阶段的含义待补充）：

![](/data/image/network/https/image-6.png)

---

Wireshark 配置 keylog 文件：

- 打开 Wireshark- 菜单 Edit → Preferences（编辑 → 首选项）
- 在左侧选择 Protocols → TLS
- 找到 (Pre)-Master-Secret log filename
- 选择你的 keys.log 文件路径


![](/data/image/network/https/image-7.png)

![](/data/image/network/https/image-8.png)