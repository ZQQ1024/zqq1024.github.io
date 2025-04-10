---
title: "ARP"
weight: 1
bookToc: true
---

# ARP实验

## 网络拓扑

- **PC1**  
  - IP 地址: `192.168.1.2` 

  - MAC 地址: `0c:37:7f:a0:00:00`   

  - OS: debian-12.6

- **PC2**  
  - IP 地址: `192.168.1.3`  
  - MAC 地址: `0c:dc:99:74:00:00`
  - OS: debian-12.6
  
- **CiscoIOSvL2 交换机**  
  - 交换机连接着 **PC1** 和 **PC2**，通过以太网连接

![](/data/image/gns3/arp/image-20250228090610292.png)

## GNS配置与操作

**PC1 配置IP**

```bash
sudo ip addr add 192.168.1.2/24 dev ens4
sudo ip link set ens4 up
```

**PC2 配置IP**

```bash
sudo ip addr add 192.168.1.3/24 dev ens4
sudo ip link set ens4 up
```

**PC1 ping PC2**

```bash
ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=1.55 ms
64 bytes from 192.168.1.3: icmp_seq=2 ttl=64 time=1.05 ms
^C
--- 192.168.1.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.046/1.297/1.549/0.251 ms
```

**2台机器上查看ARP**

```bash
debian@debian:~$ ip neighbor show
192.168.1.3 dev ens4 lladdr 0c:dc:99:74:00:00 STALE


debian@debian:~$ ip neighbor show
192.168.1.2 dev ens4 lladdr 0c:37:7f:a0:00:00 STALE

# lladdr 代表 Link Layer Address，即链路层地址，通常指的是设备的 MAC 地址。
```

![](/data/image/gns3/arp/image-20250227174839890.png)

**交换机查看mac表**

```
Switch>show mac address-table
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0c37.7fa0.0000    DYNAMIC     Gi0/0
   1    0cdc.9974.0000    DYNAMIC     Gi0/1
Total Mac Addresses for this criterion: 2
```

![](/data/image/gns3/arp/image-20250227174919358.png)


## ARP协议工作流程

**抓包分析**

![](/data/image/gns3/arp/image-20250228090922337.png)

**清空缓存并验证**

```bash
debian@debian:~$ sudo ip -s -s neigh flush all
192.168.1.3 dev ens4 lladdr 0c:dc:99:74:00:00  used 50/50/15probes 1 STALE

*** Round 1, deleting 1 entries ***
*** Flush is complete after 1 round ***

debian@debian:~$ ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=4.43 ms
64 bytes from 192.168.1.3: icmp_seq=2 ttl=64 time=1.87 ms
^C
--- 192.168.1.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.873/3.150/4.427/1.277 ms

debian@debian:~$ ip neighbor show
192.168.1.3 dev ens4 lladdr 0c:dc:99:74:00:00 STALE
```

![](/data/image/gns3/arp/image-20250227180501577.png)

**协议流程**：

1. **PC1** 发送了二层广播，询问谁知道`192.168.1.3`的mac地址

```
Source				Destination		Protoco 	Length 		Info
0c:37:7f:a0:00:00	Broadcast		ARP			42			Who has 192.168.1.3? Tell 192.168.1.2
```

2. **PC2** 向**PC1** 发送回复，告知`192.168.1.3`的mac地址为`0c:dc:99:74:00:00`

```
Source				Destination			Protoco 	Length 		Info
0c:dc:99:74:00:00	0c:37:7f:a0:00:00	ARP			60			192.168.1.3 is at 0c:dc:99:74:00:00
```

3. **PC1** 将结果保存在本地ARP缓存中，后续发送给`192.168.1.3`对应的二层报文mac地址都为`0c:dc:99:74:00:00`


## 结论

通过ARP实验理解以下几点：

- 交换机的主要作用之一就是根据 MAC 地址将数据帧转发到正确的端口，这里的端口就是指交换机上面的物理端口

  ![](/data/image/gns3/arp/image-20250227175201325.png)

- ARP (Address Resolution Protocol) 的主要作用是将 **IP 地址** 转换为对应的 **MAC 地址**，在局域网中，数据链路层（如以太网）使用的是 **MAC 地址** 来传输数据帧，而不是 **IP 地址**

- 此场景不涉及网关也能通信，网关通常用于跨网络通信，帮助路由数据到其他网络，但**PC1**和**PC2**都在同一个子网内（会选择对应的网卡为`ens4`发出），他们只需要知道对方的MAC地址即可直接交换数据
