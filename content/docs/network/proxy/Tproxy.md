---
title: "Tproxy"
weight: 3
bookToc: true
---

# Tproxy

这里说的透明代理，是指被代理的设备不需要感知运行任何代理相关的软件或进行代理相关的配置，接入网络后就被自动代理。负责代理的处理的逻辑放在设备接入的网关处，如路由器、旁路由。

介绍下面2种方案

## tun2socks

https://github.com/xjasonlyu/tun2socks

这里不多介绍，参看官方wiki

## iptables/nftables

基于 `iptables/nftables` 方案实现的透明代理相较于 `tun2socks` 拥有更好的性能。

### 背景知识

`Netfilter` 是内核提供的一个框架，在整个网络流程的若干处理环节放置了一些检测点，可在检查点注册回调函数实现如用于包过滤，nat，包修改等功能。`iptables`是基于 `Netfilter` 提供的接口，运行在用户空间，用于构建防火墙规则的命令行工具。`nftables` 是 `iptables` 的继任者，支持更多新特性和优化。

```
        +---------------------+
        | iptables / nftables | <=== User-space Tools
        +----------+----------+
                   |
        +----------v----------+
        |      Netfilter      | <=== Linux kernel
        +---------------------+
```

`Netfilter` 有5个预定义的`Netfilter` hook点，注册在这些勾子处的回调函数触发时机说明如下：
- `NF_IP_PRE_ROUTING`: 简称`PREROUTING`，数据包刚进入网络栈时触发，在路由决策之前。适合用于 NAT、透明代理等
- `NF_IP_LOCAL_IN`: 简称`INPUT`，路由判定后，确定数据包目的地是本机时触发。适合用于入站防火墙规则（如拦截特定端口）
- `NF_IP_FORWARD`: 简称`FORWARD`，路由判定后，数据包需被转发到其他主机时触发。用于网关、防火墙转发控制
- `NF_IP_LOCAL_OUT`: 简称`OUTPUT`，本机产生的出站数据包刚进入网络栈时触发（数据包刚由本地应用程序生成）。可用于修改或监控本地发出的流量
- `NF_IP_POST_ROUTING`:  简称`POSTROUTING`，所有即将离开本机的出站或转发数据包，在路由处理完成后、真正发出前触发。常用于 SNAT 或封包修改

如下图所示，不同hook点对应对包不同的处理阶段：
![](/data/image/network/netfilter-hooks.png)

---

理解完上述概念后在再回忆一下`iptables`：

`iptables` 有 `filter, nat, mangle, raw` 四种内建表。其中 `filter` 是默认表，它有三种内建链，对应上面的`INPUT`/`OUTPUT`/`FORWARD`，链下面包含一系列规则，规则主要包括匹配条件和目标动作(target)，动作包含 `ACCEPT, DROP, REJECT, DNAT, SNAT, MARK, RETURN` 等。

可以简单概括为，**表**是基于对包进行不同处理的功能划分，**链**对应`Netfilter` hook点是处理的阶段划分

iptables 表执行的优先级，本质上是指在相同 Hook 点上，内核处理各表链的先后顺序，用数值表示，数值越小，优先级越高，越早处理。各表对应的数值如下：
- `raw`: 	-300，最早执行，如用于跳过conntrack连接跟踪
- `mangle`: -150，第二执行，如修改包内容
- `nat`: 0，如地址转换，较晚执行
- `filter`: 100，	最后执行，决定 `ACCEPT/DROP` 等

如下 `iptables` 规则，往`filter`表中插入了一条规则，该规则会将来源地址为`192.168.1.0/24`且目标端口为`80`的tcp流量丢掉。相较于其他表`INPUT`链中的规则，该规则是最后执行的。
```bash
iptables -A INPUT -p tcp --dport 80 -s 192.168.1.0/24 -j DROP
```
---

### 方案思路

介绍完背景知识，正式回到基于 `iptables/nftables` 的透明代理方案

路由器作为网关，流量走向大致有以下3种情况，经过的`Netfilter` hook点如下：
1. 局域网设备通过路由器访问互联网：`局域网设备 -> PREROUTING -> FORWARD -> POSTINGROUTING -> Internet`
2. 局域网设备访问路由器：`局域网设备 -> PREROUTING -> INPUT -> 路由器`
3. 路由器访问互联网：`路由器 -> INPUT -> POSTINGROUTING -> Internet`

总体思路：通过使用 `iptables`/`nftables`的 `TPROXY` target 操控 `PREROUTING`链和`OUTPUT`链的流量走向，转发到 `v2ray` 以 `dokodemo-door` 为协议的inbound（并设置为` "tproxy": "tproxy"`，对应设置了 [`IP_TRANSPARENT`选项](https://man7.org/linux/man-pages/man7/ip.7.html) ），这允许代理进程接收原始目标地址不是它自己的数据包以及绑定非本地地址（详细说明见下），来实现代理局域网设备和网关本机

### 具体步骤

**Step 1 - v2ray配置**：
```json
{
  "dns":{
     "servers":[
        {
           "address":"223.5.5.5",
           "port":53,
           "domains":[
              "geosite:cn"
           ]
        },
        {
           "address":"tcp://1.1.1.1",
           "domains":["geosite:geolocation-!cn"]
        },
        "localhost"
     ]
  },
  "log":{
     "access":"access.log",
     "error":"error.log",
     "loglevel":"warning"
  },
  "inbounds":[
     {
        "tag":"tproxy-in",
        "port":12345,
        "protocol":"dokodemo-door",
        "settings":{
           "network":"tcp,udp",
           "followRedirect":true
        },
        "sniffing":{
           "enabled":true,
           "destOverride": ["http", "tls"]
        },
        "streamSettings":{
           "sockopt":{
              "tproxy":"tproxy"
           }
        }
     }
  ],
  "outbounds":[
     {
        "tag":"proxy",
        "protocol":"socks",
        "settings":{
           "servers":[
              {
                 "address":"127.0.0.1",
                 "port":40080
              }
           ]
        },
        "streamSettings":{
           "sockopt":{
              "mark":255
           }
        }
     },
     {
        "tag":"direct",
        "protocol":"freedom",
        "settings":{
           "domainStrategy":"UseIPv4"
        },
        "streamSettings":{
           "sockopt":{
              "mark":255
           }
        }
     },
     {
        "tag":"block",
        "protocol":"blackhole"
     },
     {
        "protocol":"dns",
        "tag":"dns-out",
        "streamSettings":{
           "sockopt":{
              "mark":255
           }
        }
     }
  ],
  "routing":{
     "domainStrategy":"IPOnDemand",
     "rules":[
        {                               
           "type":"field",              
           "outboundTag":"block",           
           "domain":[                     
              "geosite:category-ads-all"
           ],                     
           "enabled":true         
        },
        {
        "type": "field",
        "inboundTag": [
            "dns-in"
        ],
        "outboundTag": "dns-out",
        "enabled": true
        },
        {
           "type":"field",
           "inboundTag":[
              "tproxy-in"
           ],
           "port":53,
           "network":"udp",
           "outboundTag":"dns-out",
           "enabled":true
        },
        {
           "type":"field",
           "inboundTag":[
              "tproxy-in"
           ],
           "port":123,
           "network":"udp",
           "outboundTag":"direct"
        },
        {
           "type":"field",
           "ip":[
              "223.5.5.5",
              "114.114.114.114",
              "199.168.138.252"
           ],
           "outboundTag":"direct"
        },
        {
           "type":"field",
           "ip":[
              "1.1.1.1"
           ],
           "outboundTag":"proxy"
        },
        {
           "type":"field",
           "outboundTag":"direct",
           "domain":["geosite:cn"],
           "enabled":true
        },
        {
           "type":"field",
           "outboundTag":"direct",
           "ip":[
              "geoip:private",
              "geoip:cn"
           ],
           "enabled":true
        },
        {
           "type":"field",
           "outboundTag":"direct",
           "protocol":[
              "bittorrent"
           ],
           "enabled":true
        },
        {
           "type":"field",
           "port":"0-65535",
           "outboundTag":"proxy",
           "enabled":true
        }
     ]
  }
}
```

主要改动：
- 设置了`tproxy` 的 `dokodemo-door` 协议inbound用于接收重定向后的流量
- `direct`/`proxy` 添加了 `"mark":255` 的标记，结合下面 `iptables/nftables` 配置，必变透明代理的流量被重复处理
- 结合实际替换 `proxy` outbound

**Step 2 - 创建策略路由**：
```bash
ip rule add fwmark 1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100
```

定义了一个单独的路由表，当透明的代理数据包被打上 `fwmark = 1`（使用 `iptables -j MARK` 或 `nft mark set`），就会根据 `路由表 100` 进行路由查找。这里`路由表 100`定义，任何目的地址的数据包交给 loopback 回环接口处理，继而可以将流量交给用户空间的代理程序（如v2ray）处理，而不是发往外网。

{{< hint info >}}
当设置上述策略路由后，以下代码即可捕获任何发往`5301`端口的udp报文，不论目的地址是什么
```C
  Socket s(AF_INET, SOCK_DGRAM, 0);
  ComboAddress local("0.0.0.0", 5301);
  ComboAddress remote(local);

  SBind(s, local);

  for(;;) {
    string packet=SRecvfrom(s, 1500, remote);
    cout<<"Received a packet from "<<remote.toStringWithPort()<<endl;
  }
```
{{< /hint >}}

**Step 3 - nftables/iptables 配置**：

核心配置是：
```bash
iptables -t mangle -A PREROUTING -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1
```
将 tcp 流量 tproxy 到本地 12345 端口，通过设置 `IP_TRANSPARENT` socket 选项接受来自 iptables `TPROXY` 重定向的流量，并允许用户空间程序在 
**不拥有该源地址的前提下，绑定并接收/发送任意源地址的数据包**，这样网关收到来自局域网内设备的流量后，并不会修改真实来源地址，通过**看起来像伪造来源地址的方式**，实现了透明代理

{{< hint info >}}
以下代码在本机网络栈伪造并发出了一个源地址为 `1.2.3.4:5300`，目的地址为 `198.41.0.4:53` 的 UDP 包
```C
  Socket s(AF_INET, SOCK_DGRAM, 0);
  SSetsockopt(s, IPPROTO_IP, IP_TRANSPARENT, 1);
  ComboAddress local("1.2.3.4", 5300);
  ComboAddress remote("198.41.0.4", 53);
  
  SBind(s, local);
  SSendto(s, "hi!", remote);
```
{{< /hint >}}


{{< tabs "配置" >}}
{{< tab "nftables" >}}
```bash
#!/usr/sbin/nft -f

#  prerequsite
#  ip rule  add fwmark 1 table 100
#  ip route add local 0.0.0.0/0 dev lo table 100


flush ruleset

define RESERVED_IP = {
    10.0.0.0/8,
    100.64.0.0/10,
    127.0.0.0/8,
    169.254.0.0/16,
    172.16.0.0/12,
    192.0.0.0/24,
    224.0.0.0/4,
    240.0.0.0/4,
    255.255.255.255/32
}


table ip v2ray {
    # 代理局域网设备
    chain prerouting {
        type filter hook prerouting priority 0; policy accept;

        ip daddr $RESERVED_IP return
        ip saddr $RESERVED_IP return
        
        ip daddr 192.168.0.0/16 tcp dport != 53 return
        ip daddr 192.168.0.0/16 udp dport != 53 return
        
        meta mark 255 return 
        
        meta l4proto { tcp, udp } meta mark set 1 tproxy to 127.0.0.1:12345 accept # fwmark=1
    }
    
    # 代理网关本机
    chain output {
        type route hook output priority 0; policy accept;
        
        ip daddr $RESERVED_IP return
        ip saddr $RESERVED_IP return
        
        # tproxy dns
        ip daddr 192.168.0.0/16 tcp dport != 53 return
        ip daddr 192.168.0.0/16 udp dport != 53 return
        
        meta mark 255 return
        
        # OUTPUT -> lo -> PREROUTING -> TPROXY
        meta l4proto { tcp, udp } meta mark set 1 accept
    }
}

# 防止代理自己的流量再次进入循环
table ip filter {
    chain divert {
        type filter hook prerouting priority -150; policy accept;
        meta l4proto tcp socket transparent 1 meta mark set 1 accept
    }
}
```
{{< /tab >}}
{{< tab "iptables" >}}
```bash
iptables -t mangle -N V2RAY_PREROUTING

# 保留 IP 不代理
iptables -t mangle -A V2RAY_PREROUTING -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 100.64.0.0/10 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 192.0.0.0/24 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 240.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 255.255.255.255/32 -j RETURN

iptables -t mangle -A V2RAY_PREROUTING -s 10.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -s 100.64.0.0/10 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -s 127.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -s 169.254.0.0/16 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -s 172.16.0.0/12 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -s 192.0.0.0/24 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -s 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -s 240.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -s 255.255.255.255/32 -j RETURN

# 不代理局域网流量，除了DNS协议，DNS交由v2ray处理
iptables -t mangle -A V2RAY_PREROUTING -d 192.168.0.0/16 -p tcp ! --dport 53 -j RETURN
iptables -t mangle -A V2RAY_PREROUTING -d 192.168.0.0/16 -p udp ! --dport 53 -j RETURN

# 忽略 mark=255，跳过v2ray的 direct/proxy流量，避免这些流量被二次处理
iptables -t mangle -A V2RAY_PREROUTING -m mark --mark 255 -j RETURN

# 设置 mark 1 并 TPROXY 到 12345 端口
iptables -t mangle -A V2RAY_PREROUTING -p tcp -j TPROXY --on-port 12345 --tproxy-mark 1
iptables -t mangle -A V2RAY_PREROUTING -p udp -j TPROXY --on-port 12345 --tproxy-mark 1

# 应用到 PREROUTING
iptables -t mangle -A PREROUTING -j V2RAY_PREROUTING

iptables -t mangle -N V2RAY_OUTPUT
# 保留 IP
iptables -t mangle -A V2RAY_OUTPUT -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 100.64.0.0/10 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 192.0.0.0/24 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 240.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 255.255.255.255/32 -j RETURN

iptables -t mangle -A V2RAY_OUTPUT -s 10.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -s 100.64.0.0/10 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -s 127.0.0.0/8 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -s 169.254.0.0/16 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -s 172.16.0.0/12 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -s 192.0.0.0/24 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -s 224.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -s 240.0.0.0/4 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -s 255.255.255.255/32 -j RETURN

# 不代理局域网流量，除了DNS协议，DNS交由v2ray处理
iptables -t mangle -A V2RAY_OUTPUT -d 192.168.0.0/16 -p tcp ! --dport 53 -j RETURN
iptables -t mangle -A V2RAY_OUTPUT -d 192.168.0.0/16 -p udp ! --dport 53 -j RETURN

# 忽略 mark=255，跳过v2ray的 direct/proxy流量，避免这些流量被二次处理
iptables -t mangle -A V2RAY_OUTPUT -m mark --mark 255 -j RETURN

# 打 mark 1，系统会转交 PREROUTING 中的 TPROXY 逻辑
iptables -t mangle -A V2RAY_OUTPUT -p tcp -j MARK --set-mark 1
iptables -t mangle -A V2RAY_OUTPUT -p udp -j MARK --set-mark 1

# 应用到 OUTPUT
iptables -t mangle -A OUTPUT -j V2RAY_OUTPUT

# 防止代理自己的流量再次进入循环
iptables -t mangle -N DIVERT
iptables -t mangle -A DIVERT -j MARK --set-mark 1
iptables -t mangle -A DIVERT -j ACCEPT
iptables -t mangle -I PREROUTING -p tcp -m socket -j DIVERT
```
{{< /tab >}}
{{< /tabs >}}

一些说明：
- 在 `nftables` 中 `socket transparent 1` 实质就是检测 socket 是否绑定了 `IP_TRANSPARENT`，等价于 `iptables -m socket`；在 `iptables` 中，处理 socket 匹配（即 `-m socket`）只能在 `mangle` 表的 `PREROUTING` 链 中使用。在 `nftables` 中，`socket transparent 1` 只允许在 `filter` 表中使用。


## 参考链接

> https://powerdns.org/tproxydoc/tproxy.md.html  
https://xtls.github.io/document/level-2/tproxy.html  
https://guide.v2fly.org/app/tproxy.html  
https://toutyrater.github.io/app/tproxy.html