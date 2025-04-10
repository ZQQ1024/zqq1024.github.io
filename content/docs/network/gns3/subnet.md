---
title: "Subnet"
weight: 3
bookToc: true
---

# 子网划分

## 划分思路

需要将 `192.168.1.0/24` 这个网段划分为多个子网，分别分配给不同的部门，具体需求如下

- 部门A：50台
- 部门B：20台
- 部门C：10台
- 部门D：5台


划分思路：

- `192.168.1.0/24`网络范围内有`2^8=256`个IP地址，总体思路后 `x` bit用于主机地址（容纳对应数量），前`8-x` bit用于网路地址（空间划分）

- 按照主机数 `N` **从大到小**的方式分配，找到满足 `2^h - 2 ≥ N` 的最小 `h`（减去 2 是因为需要 **预留网络地址和广播地址**，或者再考虑去除一个网关地址），然后计算 **子网掩码** 位数：`32 - h` 
- 当前子网的**起始地址 + 2^h（总 IP 数）**，就是下一个子网的起始地址，通过 **+** 加号表示当前子网已经占用掉了多少个IP地址


| 部门 | 需要主机数 N | 所需最小h `2^h - 2 ≥ N` | 可用主机数 `2^h-2` | 子网掩码            | IP地址格式           | 子网网络号       |
| ---- | ------------ | ----------------------- | ------------------ | ------------------- | -------------------- | ---------------- |
| A    | 50           | 6                       | 62                 | /26 255.255.255.192 | 192.168.1.[00??????] | 192.168.1.0/26   |
| B    | 20           | 5                       | 30                 | /27 255.255.255.224 | 192.168.1.[010?????] | 192.168.1.64/27  |
| C    | 10           | 4                       | 14                 | /28 255.255.255.240 | 192.168.1.[0110????] | 192.168.1.96/28  |
| D    | 5            | 3                       | 6                  | /29 255.255.255.248 | 192.168.1.[01110???] | 192.168.1.112/29 |


虽然逻辑上来看不同部门的子网都属于`192.168.1.0/24`范围内，但从IP层来看依然属于不同的网络，需要接入路由器进行通讯

![](/data/image/gns3/subnet/image-20250228150945868.png)

## GNS配置与操作

这里4台PC分别模拟代表4个部门

**A**

```bash
sudo ip addr add 192.168.1.2/26 dev ens4
sudo ip link set ens4 up
sudo ip route add default via 192.168.1.1
```

**B**

```bash
sudo ip addr add 192.168.1.66/27 dev ens4
sudo ip link set ens4 up
sudo ip route add default via 192.168.1.65
```

**C**

```bash
sudo ip addr add 192.168.1.98/28 dev ens4
sudo ip link set ens4 up
sudo ip route add default via 192.168.1.97
```

**D**

```bash
sudo ip addr add 192.168.1.114/29 dev ens4
sudo ip link set ens4 up
sudo ip route add default via 192.168.1.113
```

**路由器**

```
show ip interface brief
Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0         unassigned      YES unset  administratively down down
GigabitEthernet0/1         unassigned      YES unset  administratively down down
GigabitEthernet0/2         unassigned      YES unset  administratively down down
GigabitEthernet0/3         unassigned      YES unset  administratively down down

enable
configure terminal

interface GigabitEthernet0/0 
ip address 192.168.1.1 255.255.255.192  # A网关
no shutdown

interface GigabitEthernet0/1
ip address 192.168.1.65 255.255.255.224  # B网关
no shutdown

interface GigabitEthernet0/2
ip address 192.168.1.97 255.255.255.240  # C网关
no shutdown

interface GigabitEthernet0/3
ip address 192.168.1.113 255.255.255.248  # D网关
no shutdown

exit
```

**ping测试**

```bash
debian@debian:~$ ping 192.168.1.66
PING 192.168.1.66 (192.168.1.66) 56(84) bytes of data.
64 bytes from 192.168.1.66: icmp_seq=1 ttl=63 time=6.87 ms
^C
--- 192.168.1.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 6.865/6.865/6.865/0.000 ms
debian@debian:~$ ping 192.168.1.98
PING 192.168.1.98 (192.168.1.98) 56(84) bytes of data.
64 bytes from 192.168.1.98: icmp_seq=1 ttl=63 time=5.35 ms
^C
--- 192.168.1.98 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 5.348/5.348/5.348/0.000 ms
debian@debian:~$ ping 192.168.1.114
PING 192.168.1.114 (192.168.1.114) 56(84) bytes of data.
64 bytes from 192.168.1.114: icmp_seq=1 ttl=63 time=4.44 ms
^C
--- 192.168.1.114 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 4.438/4.438/4.438/0.000 ms
```

![](/data/image/gns3/subnet/image-20250228151749441.png)

## 总结

- 子网划分思路见上
- 不同子网依然属于不同的网络，需要接入路由器进行通讯
