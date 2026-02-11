---
title: "xcap分层"
weight: 1
bookToc: true
---

[xcap](https://xcap.weebly.com/) 是一个免费的网络发包工具，可以使用xcap制造和发送报文，可用于DoS打流等

我们通过这个软件做网络分层实验，手动创建每层报文，理解 `TCP/IP` 模型中每一层的作用，直观感受每层 PDU(protocol data unit) 应包含的字段和层层包含的关系

![](/data/image/network/xcap/image-12.png)

**目标：往ip地址为 `199.168.138.252` 的目标发送 `80/tcp` 端口 `http` 报文**

前置条件，先创建报文组和报文

![](/data/image/network/xcap/image.png)

![](/data/image/network/xcap/image-1.png)

### 1 链路层 Ethernet

这一层PDU为Frame，包含：源 MAC、目的 MAC、EtherType、Packet

![](/data/image/network/xcap/image-2.png)

![](/data/image/network/xcap/image-3.png)

通过路由表和目的地址 `199.168.138.252` 综合判断，发给目的IP为 `199.168.138.252` 的报文会通过网卡`Intel(R) Ethernet Connection (7) I219-LM`发出，来源IP为 `192.168.18.19` ，并通过网关 `192.168.18.254` 发到外部网络


以太网（链路层）通过 MAC 地址进行通讯，并不感知IP地址：

- 同网段 → 目标主机的 MAC
- 跨网段 → 网关的 MAC

网关的MAC地址为 `00:00:5E:00:01:12`

![](/data/image/network/xcap/image-4.png)

本机网卡的MAC地址为`00:D8:61:90:9A:92`

所以，此时构造`Ethernet`报文填写的信息如下：

![](/data/image/network/xcap/image-5.png)


### 2 网络层 IPv4

这一层PDU为Packet，关键字段：来源 IP、目的 IP、TTL、协议号、Segment/Datagram

来源IP为`192.168.18.19`

目的IP为`199.168.138.252`

![](/data/image/network/xcap/image-6.png)


### 3 传输层 TCP

这一层PDU为Segment(TCP)/Datagram(UDP)，TCP关键字段：源端口、目的端口、SEQ、ACK、Flags标志位、应用层报文

![](/data/image/network/xcap/image-7.png)

### 4 应用层 HTTP

常见协议：HTTP，最简单的HTTP报文`GET / HTTP/1.0\r\n\r\n`

![](/data/image/network/xcap/image-8.png)


### 5 对端抓包

启动网卡，选择网卡，发包

![](/data/image/network/xcap/image-9.png)

![](/data/image/network/xcap/image-10.png)

![](/data/image/network/xcap/image-11.png)

来源IP为`223.108.79.97`与`192.168.18.19`不一致，端口`15387`也与源端口不一致，但`SEQ`与来源是一致的，是因为中间有`NAT`转发