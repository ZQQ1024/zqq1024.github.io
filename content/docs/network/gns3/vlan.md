---
title: "VLAN + Trunk"
weight: 4
bookToc: true
---

## 广播域

**广播域（Broadcast Domain）** 是指可以接收同一个广播帧的所有设备的集合。二层交换机默认不分隔广播域，所有连接的设备都会收到广播。当网络规模扩大时，会带来 **性能、管理、安全性** 等方面的问题。

![](/data/image/gns3/vlan/image-20250228171522760.png)

使用GNS自带的VPCS，交换机为`vIOS L2`

```
PC1> ip 192.168.10.1/24
Checking for duplicate address...
PC1 : 192.168.10.1 255.255.255.0

PC2> ip 192.168.10.2/24
Checking for duplicate address...
PC2 : 192.168.10.2 255.255.255.0

PC3> ip 192.168.20.1/24
Checking for duplicate address...
PC3 : 192.168.20.1 255.255.255.0

PC4> ip 192.168.20.2/24
Checking for duplicate address...
PC4 : 192.168.20.2 255.255.255.0
```


**PC1 ping PC2**，同时在PC4抓包

![](/data/image/gns3/vlan/image-20250228172152715.png)

```
# （ping PC3需要网关）
PC1> clear arp

PC1> ping 192.168.10.2

84 bytes from 192.168.10.2 icmp_seq=1 ttl=64 time=2.357 ms
84 bytes from 192.168.10.2 icmp_seq=2 ttl=64 time=1.646 ms
84 bytes from 192.168.10.2 icmp_seq=3 ttl=64 time=1.783 ms
84 bytes from 192.168.10.2 icmp_seq=4 ttl=64 time=1.806 ms
84 bytes from 192.168.10.2 icmp_seq=5 ttl=64 time=1.912 ms
```

发现PC4也能收到ARP广播

![](/data/image/gns3/vlan/image-20250228172321775.png)


## VLAN 隔离广播域

 **VLAN**主要是交换机（Switch）的功能，但不仅限于交换机。在交换机上，VLAN 通过**端口绑定**的方式实现，交换机的每个端口可以分配到**不同的 VLAN**。交换机VLAN端口有2种模式，其中Trunk下文再说：

- Access 端口：主要用于连接 PC、服务器等终端设备，只能属于**一个 VLAN**。PC段不感知 VLAN Tag，Tag的处理都在交换机侧进行
- Trunk 端口：主要用于连接交换机、路由器等，允许**多个 VLAN 通过**，使用 **802.1Q** 添加 VLAN Tag，连接双方都要感知VLAN Tag


引入 VLAN（虚拟局域网，Virtual LAN）隔离广播域，不同 VLAN 之间默认不互通，每个 VLAN 都是**独立的广播域**。VLAN 内部的广播会**被限制在 VLAN 内部**，不会扩散到其他 VLAN。VLAN **基于逻辑划分网络**，没有 VLAN 时，网络是基于物理拓扑管理的，改变广播域涉及打破物理位置、重新布线等。

![](/data/image/gns3/vlan/image-20250228173249659.png)


将`PC1`和`PC2`放到一个广播域，VLAN为10，使用的交换机端口是`Gi0/0`和`Gi0/1`，将`PC3`和`PC4`放到另一个广播域，VLAN为20，使用的交换机端口是`Gi0/2`和`Gi0/3`。


**仅需在交换机处配置**

```
Switch>show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     unassigned      YES unset  up                    up
GigabitEthernet0/1     unassigned      YES unset  up                    up
GigabitEthernet0/2     unassigned      YES unset  up                    up
GigabitEthernet0/3     unassigned      YES unset  up                    up
GigabitEthernet1/0     unassigned      YES unset  down                  down
GigabitEthernet1/1     unassigned      YES unset  down                  down
GigabitEthernet1/2     unassigned      YES unset  down                  down
GigabitEthernet1/3     unassigned      YES unset  down                  down
GigabitEthernet2/0     unassigned      YES unset  down                  down
GigabitEthernet2/1     unassigned      YES unset  down                  down
GigabitEthernet2/2     unassigned      YES unset  down                  down
GigabitEthernet2/3     unassigned      YES unset  down                  down
GigabitEthernet3/0     unassigned      YES unset  down                  down
GigabitEthernet3/1     unassigned      YES unset  down                  down
GigabitEthernet3/2     unassigned      YES unset  down                  down
GigabitEthernet3/3     unassigned      YES unset  down                  down

Switch>enable
Switch>configure terminal

Switch(config)#interface GigabitEthernet0/0
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 10
% Access VLAN does not exist. Creating vlan 10
Switch(config-if)#no shutdown

Switch(config-if)#interface GigabitEthernet0/1
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 10
Switch(config-if)#no shutdown

Switch(config-if)#interface GigabitEthernet0/2
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 20
% Access VLAN does not exist. Creating vlan 20
Switch(config-if)#no shutdown

Switch(config-if)#interface GigabitEthernet0/3
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 20
Switch(config-if)#no shutdown
```

```
Switch#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi1/0, Gi1/1, Gi1/2, Gi1/3
                                                Gi2/0, Gi2/1, Gi2/2, Gi2/3
                                                Gi3/0, Gi3/1, Gi3/2, Gi3/3
10   VLAN0010                         active    Gi0/0, Gi0/1
20   VLAN0020                         active    Gi0/2, Gi0/3
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup
1004 fddinet-default                  act/unsup
1005 trnet-default                    act/unsup
```

![](/data/image/gns3/vlan/image-20250228174856840.png)

```
Switch#show interfaces switchport
Name: Gi0/0
Switchport: Enabled
Administrative Mode: static access
...
```

![](/data/image/gns3/vlan/image-20250228175253031.png)


**继续 PC1 ping PC2**，同时在PC2和PC4抓包

```
PC1> clear arp

PC1> ping 192.168.10.2

84 bytes from 192.168.10.2 icmp_seq=1 ttl=64 time=1.755 ms
84 bytes from 192.168.10.2 icmp_seq=2 ttl=64 time=2.938 ms
84 bytes from 192.168.10.2 icmp_seq=3 ttl=64 time=2.507 ms
84 bytes from 192.168.10.2 icmp_seq=4 ttl=64 time=2.892 ms
84 bytes from 192.168.10.2 icmp_seq=5 ttl=64 time=3.278 ms
```

PC1和PC2在同一VLAN 10 划分的广播域，发现PC2能收到ARP广播

PC1和PC4不在同一个广播域，发现PC4不能收到ARP广播

![](/data/image/gns3/vlan/image-20250228175942823.png)

## Trunk

"trunk" 一词表示“树干”、“主干道”，在VLAN中trunk允许多个 VLAN 的数据包在同一物理链路上共存并传输，而不是每个 VLAN 都需要单独的物理连接。这样的设置大大提高了网络的灵活性和效率。


如下网络拓扑，不同区域的机器使用相同的VLAN，中间连接的交换机可以使用trunk，减少物理链接。

![](/data/image/gns3/vlan/image-20250303093133059.png)

**PC5**

```
PC5> ip 192.168.10.1/24
Checking for duplicate address...
PC5 : 192.168.10.1 255.255.255.0
```

**PC7**

```
PC7> ip 192.168.10.2/24
Checking for duplicate address...
PC7 : 192.168.10.2 255.255.255.0
```

**PC6**

```
PC6> ip 192.168.20.1/24
Checking for duplicate address...
PC6 : 192.168.20.1 255.255.255.0
```

**PC8**

```
PC8> ip 192.168.20.2/24
Checking for duplicate address...
PC8 : 192.168.20.2 255.255.255.0
```

**SW1**

```
show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     unassigned      YES unset  up                    up
GigabitEthernet0/1     unassigned      YES unset  up                    up
GigabitEthernet0/2     unassigned      YES unset  up                    up

Switch>enable
Switch#configure terminal
Switch(config)#interface GigabitEthernet0/0
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 10
% Access VLAN does not exist. Creating vlan 
Switch(config-if)#no shutdown

Switch(config)#interface GigabitEthernet0/1
Switch(config-if)#switchport mode access
Switch(config-if)#switchport access vlan 20
% Access VLAN does not exist. Creating vlan 
Switch(config-if)#no shutdown

Switch(config)#interface GigabitEthernet0/2
Switch(config-if)#switchport trunk encapsulation dot1q
Switch(config-if)#switchport mode trunk
Switch(config-if)#switchport trunk allowed vlan 10,20
Switch(config-if)#no shutdown

Switch>show vlan brief
Switch>show interfaces switchport
Switch>show interfaces trunk
```

**SW2** 同上

PC5 ping PC7，PC7 和 PC8 抓包

![](/data/image/gns3/vlan/image-20250303100113521.png)

PC7 和 PC5在用一个VLAN，可以收到广播，PC8和PC5不在同一个VLAN，无法收到广播

802.1Q协议也很简单，就是交换机层面加上一个tag

![](/data/image/gns3/vlan/image-20250303100517642.png)