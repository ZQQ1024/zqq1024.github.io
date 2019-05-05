---
title: "Wireshark理解TCP"
categories:
  - basic
tags:
  - tcp
classes: wide

excerpt: "Wireshark抓包理解TCP"
---

使用Wireshark观察TCP三次握手和四次挥手，过程如下：
1. Firefox浏览器访问 http://39.108.7.206/accounts/login/?next=/
2. 使用Wireshark抓包，理解TCP三次握手等过程

Wireshark版本：Wireshark 2.6.6

# 1 Connection establishment 
`Connection establishment`是`Three-way handshake`

三次握手，这种翻译感觉是进行了三次握手，实际是一次握手中有three-way，即`SYN, SYN/ACK, ACK`

TCP使用`Flags`（一堆1-bit的boolean值）来控制连接的状态

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190125132313.png)

这里主要关注以下`Flag`：
- SYN - (Synchronize) Initiates a connection
- FIN - (Final) Cleanly terminates a connection
- ACK - Acknowledges received data

TCP是双向通讯，通讯双方使用`Sequence Number`（随机数）来记录自己已发送了多少数据。发送SYN或FIN包不含实际的`Payload`，大小计为1。接收方使用对方的`Sequence Number`的数值作为`Acknowledgment Number`来承认已经接收到的数据

`Three-way handshake`就是双方同步`Sequence Number`的过程，下面以Alice和Bob进行三次握手为例：

```
1 Alice ---> Bob    SYNchronize with my Initial Sequence Number of x
2 Alice <--- Bob    I received your syn, I ACKnowledge that I am ready for [x+1]
3 Alice <--- Bob    SYNchronize with my Initial Sequence Number of y
4 Alice ---> Bob    I received your syn, I ACKnowledge that I am ready for [y+1]
```
将上述2和3的`SYN`和`ACK` OR在一起，即以下：
```
1 Bob <--- Alice         SYN
2 Bob ---> Alice       SYN ACK 
3 Bob <--- Alice         ACK    
```

得到经典的`Three-way handshake`：  
![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190125160252.png)

# 2 Connection termination
`Connection termination`一般是`Four-way handshake`，因为通讯的双方各自断开是`independent`的，链接是可以`half-open`的，即自己termination对方没有termination，termination方仍然可以接收对方发的数据

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190125163641.png)（发起方不定）

```
1 Bob ---> Alice         FIN
2 Bob <--- Alice         ACK 
3 Bob <--- Alice         FIN  
4 Bob ---> Alice         ACK
```
2和3可以合并

# 3 Wireshark example

Firefox浏览器访问 http://39.108.7.206/accounts/login/?next=/ ，获取首页，然后关闭浏览器。同时，使用Wireshark抓包获取整个过程中的网络数据

Wireshark抓取的结果保存在附件，过滤条件为`ip.addr == 39.108.7.206 && tcp.port == 35246`。通过`Wireshark-Statistics-Flow Graph-TCP Flows`打开如下数据流图：

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20190125165046.png)

访问整个网站首页的过程，一共包含10个TCP segment，从上往下依次计为0-9，详细说明：
- `Segment 0`: 客户端初始化的Seq为`0f 67 c5 c5`（相对0），下一个Seq为 + 1，Ack为0
- `Segment 1`: 服务器端初始化的Seq为`f4 e7 3c 4d`（相对0），下一个Seq为 + 1，Ack为`0f 67 c5 c5` + 1
- `Segment 2`: 客户端Ack为 `f4 e7 3c 4d` + 1
- `Segment 3`: 客户端发起http GET请求，http请求的大小为470 bytes，此时Seq为`0f 67 c5 c5` + 1，下一个为 + 471
- `Segment 4`: 服务器端收到请求，先不直接返回http response，而是返回Ack，Seq为`f4 e7 3c 4d` + 1，Ack为 `0f 67 c5 c5` + 471
- `Segment 5`: 服务器端返回http response，Seq为`f4 e7 3c 4d` + 1，响应体大小为4283 bytes，下一个Seq为`f4 e7 3c 4d` + 4284
- `Segment 6`: 客户端返回Ack，Seq为`0f 67 c5 c5` + 471，Ack为`f4 e7 3c 4d` + 4284
- `Segment 7`: 客户端发送FIN，Seq为`0f 67 c5 c5` + 471，Ack为`f4 e7 3c 4d` + 4284，下一个Seq为`0f 67 c5 c5` + 472
- `Segment 8`: 服务器端返回FIN,ACK，Seq为`f4 e7 3c 4d` + 4284，ACK为`0f 67 c5 c5` + 472，下一个Seq为`f4 e7 3c 4d` + 4285
- `Segment 9`: 客户端返回ACK，Seq为`0f 67 c5 c5` + 472，ACK为`f4 e7 3c 4d` + 4285

# 4 References
[https://www.thegeekstuff.com/2012/07/wireshark-filter/  
https://en.wikipedia.org/wiki/Transmission_Control_Protocol](https://www.thegeekstuff.com/2012/07/wireshark-filter/  
https://en.wikipedia.org/wiki/Transmission_Control_Protoco)  
[http://packetlife.net/blog/2010/jun/7/understanding-tcp-sequence-acknowledgment-numbers/](http://packetlife.net/blog/2010/jun/7/understanding-tcp-sequence-acknowledgment-numbers/)  
[https://networkengineering.stackexchange.com/questions/24068/why-do-we-need-a-3-way-handshake-why-not-just-2-way  
http://enggedu.com/tamilnadu/university_questions/question_answer/be_nd_2007/5th_sem/cse/CS1302/part_b/14a_2.html](https://networkengineering.stackexchange.com/questions/24068/why-do-we-need-a-3-way-handshake-why-not-just-2-wa)  
[https://osqa-ask.wireshark.org/questions/43717/should-there-be-4-packets-at-the-end-of-tcp-communication](https://osqa-ask.wireshark.org/questions/43717/should-there-be-4-packets-at-the-end-of-tcp-communication)  