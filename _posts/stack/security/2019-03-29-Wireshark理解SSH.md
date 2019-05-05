---
title: "Wireshark理解SSH"
categories:
  - security
tags:
  - SSH
classes: wide

excerpt: "使用抓包Wireshark理解SSH"
---


使用Wireshark观察SSH连接，过程如下：

ssh登陆一台之前从未登陆过的服务器，使用Wireshark抓包，了解SSH连接过程

Wireshark版本：Wireshark 2.6.6

# 1 SSH介绍

SSH协议全称Secure Shell，是最常见的、安全连接远程服务器的协议

与Telnet最为直观的不同是ssh传输的数据是加密的

SSH中涉及到了，对称加密，非对称加密和摘要算法

# 2 Wireshark抓包

先登陆一台服务器（目标ip 39.108.7.206)
```
$ ssh root@product
```

由于是第一次登陆这台服务器，会有如下提示：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190329132357.png)  
显示了39.108.7.206服务器ECDSA公钥的fingerprint

Wireshark截图如下：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190329132756.png)

输入yes之后，指纹信息被存在`.ssh/known_hosts`：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190329133441.png)

Wireshark截图如下：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190329133657.png)

可以发现在这个时候，客户端和服务器之间的所有数据已经是加密了的`Encrypted packet`，而且计算了MAC，保证了数据的完整性和来源有效

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190329133943.png)

# 3 过程分析

ssh主要可以分为两个过程：协商和认证

首先是TCP三次握手，这个不多说

## 3.1 协商

1. Client和Server交换SSH版本信息，如上客户端版本为`SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.8`，服务器版本为`SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.4`

2. Client和Server交换密钥算法等，
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190329134730.png)  
如上主要交换如下算法信息：
    - `kex_algorithms`：密钥交换算法
    - `server_host_key_algorithms`：服务器密钥是什么算法类型的，ECDSA、RSA等
    - `encryption_algorithms`：加密算法，对数据进行加密
    - `mac_algorithms`：MAC算法
    - `compression_algorithms`：压缩算法

    最终的选择策略是，选择客户端列表中第一个受服务器端支持的，这也是我们看到ECDSA公钥的fingerprint而不是RSA公钥的fingerprint的原因
    
3. DH密钥协商：DH可以在双方不预知相关数据的前提下，在非安全信道上，平等地参与会话密钥的生成：
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190329141107.png)

4. 服务器接收到客户端发的public key，然后也把自己的public key发给客户端，同时自己生成了session key，至此所有数据开始加密传输，然后将ECDSA公钥的fingerprint加密、计算MAC发给客户端
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190329141819.png)

## 3.2 认证

这里说的认证，指的是服务器**对客户端的认证**，认证是在协商之后的原因是，若先进行认证，即使用服务器端的公钥进行加密认证数据太expensive

认证方式主要有以下两种：
- `password authentication`：传统的密码认证方式
- `ssh keypairs authentication`：使用公私密钥对进行认证

`password`也是作为基本的数据，由上面协商出来的session key进行加密、MAC传给服务器端，服务器端结合`/etc/shadow`中的盐值计算HASH值，比对密码是否正确

`keypairs`方式，客户端的公钥要导入`.ssh/authorized_keys`当中，然后客户端签名，服务器端验签

推荐使用`keypairs`的方式，毕竟`key space`要比`password`方式大很多，且若使用常用密码，存在被爆破的可能

# 4 参考链接

> [https://www.digitalocean.com/community/tutorials/understanding-the-ssh-encryption-and-connection-process](https://www.digitalocean.com/community/tutorials/understanding-the-ssh-encryption-and-connection-process)  
[https://juejin.im/post/5baaf517e51d453df0442dce](https://juejin.im/post/5baaf517e51d453df0442dce)  
[https://stackoverflow.com/questions/10060530/what-command-do-i-use-to-see-what-the-ecdsa-key-fingerprint-of-my-server-is](https://stackoverflow.com/questions/10060530/what-command-do-i-use-to-see-what-the-ecdsa-key-fingerprint-of-my-server-is)  
[https://superuser.com/questions/1369611/why-produce-a-shared-session-key-before-authentication-in-ssh](https://superuser.com/questions/1369611/why-produce-a-shared-session-key-before-authentication-in-ssh)
