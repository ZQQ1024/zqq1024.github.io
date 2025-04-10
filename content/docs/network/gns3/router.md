---
title: "Router"
weight: 1
bookToc: true
---

# 网络路由实验

## 网络拓扑

- **PC1** 
  - IP 地址: `192.168.1.100/24` 
- **PC2** 
  - IP 地址: `192.168.2.100/24` 
- **Router**
  - Cisco 7200路由器
  - `eth0` IP地址：`192.168.1.1/24`，MAC地址`ca:01:05:24:00:00`
  - `eth1` IP地址：`192.168.2.1/24`，MAC地址`ca:01:05:24:00:1c`

![](/data/image/gns3/router/image-20250228103755047.png)

## GNS配置与操作

**PC1**

```bash
sudo ip addr add 192.168.1.100/24 dev ens4
sudo ip link set ens4 up
sudo ip route add default via 192.168.1.1

# Error: Nexthop has invalid gateway.
# 网卡需要先up，另外不在一个网段也会报错。单纯的配置，不会因为1.1不通而无法配置
```

![](/data/image/gns3/router/image-20250228095159046.png)

**PC2**

```bash
sudo ip addr add 192.168.2.100/24 dev ens4
sudo ip link set ens4 up
sudo ip route add default via 192.168.2.1
```

**Router**

```bash
R1#show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            unassigned      YES unset  administratively down down
FastEthernet1/0            unassigned      YES unset  administratively down down

R1#configure terminal
R1(config)#interface FastEthernet0/0
R1(config-if)#ip address 192.168.1.1 255.255.255.0
R1(config-if)#no shutdown
*Feb 27 14:13:46.143: %LINK-3-UPDOWN: Interface FastEthernet0/0, changed state to up
*Feb 27 14:13:47.143: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/0,
changed state to up

R1(config-if)#interface FastEthernet1/0
R1(config-if)#ip address 192.168.2.1 255.255.255.0
R1(config-if)#no shutdown
```

![](/data/image/gns3/router/image-20250228095727920.png)


**PC1 ping PC2**

```
debian@debian:~$ ping 192.168.2.100
PING 192.168.2.100 (192.168.2.100) 56(84) bytes of data.
64 bytes from 192.168.2.100: icmp_seq=1 ttl=63 time=52.4 ms
64 bytes from 192.168.2.100: icmp_seq=2 ttl=63 time=16.8 ms
64 bytes from 192.168.2.100: icmp_seq=3 ttl=63 time=17.7 ms
^C
--- 192.168.2.100 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2005ms
rtt min/avg/max/mdev = 16.758/28.966/52.413/16.583 ms
```

![](/data/image/gns3/router/image-20250228095928164.png)


## 路由流程

**PC1 ping PC2** 时抓包，分析路由的工作流程如下：

**Step 1 确定next hop网关**


![](/data/image/gns3/router/image-20250228100841573.png)

基于目标地址`192.168.2.100`和路由表来决定下一跳网关地址

```bash
debian@debian:~$ ip route
default via 192.168.1.1 dev ens4
192.168.1.0/24 dev ens4 proto kernel scope link src 192.168.1.100

debian@debian:~$ ip route get 192.168.2.100
192.168.2.100 via 192.168.1.1 dev ens4 src 192.168.1.100 uid 1000
    cache
```

- `192.168.2.100`不在网络`192.168.1.0/24`内
- **最终命中** `default`默认路由，确定网关地址为 `192.168.1.1`

![](/data/image/gns3/router/image-20250228101637814.png)


**Step 2 报文发送给网关**

IP报文目的IP地址`192.168.2.100`不变，但对应二层报文的MAC地址为网关地址`192.168.1.1`的MAC地址`ca:01:05:24:00:00`

```bash
debian@debian:~$ sudo ip -s -s neigh flush all
192.168.1.1 dev ens4 lladdr ca:01:05:24:00:00  used 1215/1215/1180probes 4 STALE

*** Round 1, deleting 1 entries ***
*** Flush is complete after 1 round ***

debian@debian:~$ sudo ip neighbor show
```

![](/data/image/gns3/router/image-20250228101940598.png)

```bash
debian@debian:~$ ping 192.168.2.100
PING 192.168.2.100 (192.168.2.100) 56(84) bytes of data.
64 bytes from 192.168.2.100: icmp_seq=1 ttl=63 time=23.4 ms
64 bytes from 192.168.2.100: icmp_seq=2 ttl=63 time=10.9 ms
64 bytes from 192.168.2.100: icmp_seq=3 ttl=63 time=21.3 ms
^C
--- 192.168.2.100 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 10.908/18.533/23.371/5.456 ms
```

![](/data/image/gns3/router/image-20250228102354175.png)


**Step 3 路由器内部转发**

基于目的网络地址选择合适的网卡发出，目的地址`192.168.2.100`在`192.168.2.0/24`网络内，从`FastEthernet1/0` 网卡发出，对应MAC地址为`ca:01:05:24:00:1c`

```bash
R1#show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override

Gateway of last resort is not set

      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, FastEthernet0/0
L        192.168.1.1/32 is directly connected, FastEthernet0/0
      192.168.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.2.0/24 is directly connected, FastEthernet1/0
L        192.168.2.1/32 is directly connected, FastEthernet1/0
```

![](/data/image/gns3/router/image-20250228102640294.png)

![](/data/image/gns3/router/image-20250228102827414.png)

![](/data/image/gns3/router/image-20250228103142323.png)

## 结论

- 跨网络通讯是通过路由器逐跳转发数据包，最终到达目标地址的过程。它类似于快递运输，源 IP 和目标 IP 相当于寄件人和收件人的地址，而路由器（网关）则是不同快递公司的交接站，确保包裹可以从一个区域传送到另一个区域。在这个过程中，每一跳的 MAC 地址都会被重新封装，因为每个网段的链路层地址都是独立的，而 IP 地址始终保持不变，以确保正确传输到最终目的地。


