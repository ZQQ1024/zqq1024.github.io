---
title: "Wireshark理解SSL"
categories:
  - security
tags:
  - ssl
classes: wide

excerpt: "OP-TEE在rpi3上的移植"
---

使用Wireshark观察SSL握手（单向认证），过程如下：
1. Firefox浏览器访问 https://wiki.chislab.com  
2. 使用Wireshark抓包，理解SSL握手等过程

Wireshark版本：Wireshark 2.6.6

# 1 TLS/SSL介绍

TLS/SSL可以理解为同一产物的不同时期。TLS1.0由SSL3.0演变而来，SSL3.0已经认为是不安全的了

TLS/SSL在网络协议中位于Application Layer下，Transport Layer(TCP)上

TCP握手、SSL握手和应用数据传输的大致流程如下：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190128130353.png)

说明如下：
- TLS/SSL握手之前，要先建立TCP连接
- TLS/SSL主要分为2层，Handshake Protocol Layer和Record Protocol Layer，一层主要确定交换Record的流程规范，一层主要确定Record的格式
- 任何类型的Record(Handshake,ChangeCipherSpec,Application)，都可以进行完整性、机密性的保护，以及数据分片、填充、压缩等

# 2 Wireshark抓包
Wireshark抓包的结果保存在了附件，过滤条件为`ip.addr == 47.52.42.166 && ssl && tcp.port == 42928`

Flow图如下（不含TCP，只有TLS/SSL）：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190128112742.png)

# 3 过程分析

#### 3.1 Client Hello
Handshake Protocol: Client Hello主要包含如下信息:
- 类别: Handshake Protocol
- Version: 客户端支持的最高TLS/SSL版本
- Random: 客户端32-byte随机数，用于生成Master Secret
- Session ID: 标识Session的ID，指定session resumption的session
- Cipher Suites: 受客户端支持的加密套件（很多个suites)
- Compression Method: 客户端支持的压缩算法

关于`Cipher Suites`，以`TLS_RSA_WITH_3DES_EDE_CBC_SHA (0x000a)`为例，
- TLS: 协议版本
- RSA: 用于密钥交换算法对来源的认证
- 3DES_EDE_CBC: 数据加密算法
- SHA-1: 用于MAC

#### 3.2 Server Response
Handshake Protocol: Server Hello主要包含如下信息:
- 类别: Handshake Protocol
- Version: 客户端和服务器端都支持的最高TLS/SSL版本
- Random: 服务器端32-byte随机数，用于生成Master Secret
- Session ID: 标识Session的ID，用于session resumption
- Cipher Suites: 服务器端选择出的最安全的、客户端和服务器端都支持的一个加密套件(就1个)
- Compression Method: 客户端和服务器端都支持的压缩算法

关于Session ID标识了session，客户端标识session，客户端在Client Hello包含session ID，指明想要恢复的session状态(共享密钥)，服务器端如果找到cache匹配，则在Server Hello中返回同样的session ID，然后跳过握手步骤。服务器端也可以返回一个新的session ID（标识本次）或者空

Handshake Protocol: Certificate  
返回X.509格式的证书

Handshake Protocol: Server Key Exchange  
DH密钥交换算法服务器端的信息，使用服务器端的私钥签了名的，因为客户端对服务器端有验证

Handshake Protocol: Server Hello Done  
Server Hello结束

#### 3.3 Client Response to Server
Handshake Protocol: Client Key Exchange  
DH密钥交换算法客户端的信息

Change Cipher Spec Protocol: Change Cipher Spec  
告诉服务器端，之后的数据为加密、认证的（即Record Layer开始使用对称密钥了）

Handshake Protocol: Finished
加密之前所有Handshake Messages的hansh和MAC值，服务器端要验证

#### 3.4 Sever Response to Client
Change Cipher Spec Protocol: Change Cipher Spec  
告诉客户端，之后的数据为加密、认证的（即Record Layer开始使用对称密钥了）

Handshake Protocol: Finished  
加密之前所有Handshake Messages的hash和MAC值，客户端要验证

#### 3.5 Application Data Flow
应用数据也是加密和认证的

# 4 References
[https://en.wikipedia.org/wiki/Transport_Layer_Security](https://en.wikipedia.org/wiki/Transport_Layer_Security)  
 [https://blog.csdn.net/wangsiman/article/details/80220665](https://blog.csdn.net/wangsiman/article/details/80220665)  
 [https://security.stackexchange.com/questions/188495/what-is-the-session-id-parameter-indicate-in-client-hello-and-server-hello-messa](https://security.stackexchange.com/questions/188495/what-is-the-session-id-parameter-indicate-in-client-hello-and-server-hello-messa)  
 [https://security.stackexchange.com/questions/172825/stealing-ssl-session-id-can-cause-any-harm](https://security.stackexchange.com/questions/172825/stealing-ssl-session-id-can-cause-any-harm)    
[https://stackoverflow.com/questions/19939247/ssl-session-tickets-vs-session-ids](https://stackoverflow.com/questions/19939247/ssl-session-tickets-vs-session-ids)  

[https://blog.csdn.net/mrpre/article/details/77868669](https://blog.csdn.net/mrpre/article/details/77868669)  