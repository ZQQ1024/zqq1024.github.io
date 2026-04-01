---
title: "nmap半开放扫描"
weight: 1
bookToc: true
---

以下命令为`nmap 半开放扫描`，需要**管理员权限**执行，具有执行速度快的特点

```bash
nmap -sS -p80,888,8888 154.230.x.x
```

![](/data/image/network/nmap/image-2.png)

![](/data/image/network/nmap/image.png)

---

这里介绍一下TCP Flags，TCP header里面有6个bits表示TCP Flags，可以理解告示对端的控制信号

![](/data/image/network/nmap/image-1.png)

- 当nmap收到`tcp.flags.ack`时，会认为port open，如上图中的80端口

![](/data/image/network/nmap/image-3.png)

- 当nmap收到`tcp.flags.rst`时，会认为port closed，如上图中的888端口

![](/data/image/network/nmap/image-4.png)

- 当nmap未收到回包，会认为port filtered，可能被防火墙过滤了，如上图中的8888端口

![](/data/image/network/nmap/image-5.png)


> 可以在目的端，使用iptables控制特定端口的tcp flags返回
>
> - 对特定端口的连接请求直接返回 RST（复位包）：`iptables -A INPUT -p tcp --dport 888 -j REJECT --reject-with tcp-reset`
> - 对特定端口直接丢弃`iptables -A INPUT -p tcp --dport 8888 -j DROP`
> - **云上云主机的安全组默认是DROP的**；默认主机层面没有监听会返回RST，RST 是 TCP 协议的标准行为：告诉对方 “这里没有监听服务”

---

![](/data/image/network/nmap/image-6.png)

![](/data/image/network/nmap/image-7.png)

---

这里我们引出**TCP状态机**的概念：TCP 状态机（TCP State Machine） 是描述 TCP 连接在不同阶段的 状态变化和转换规则的模型。它定义了一个 TCP 连接从创建到关闭全过程的状态、事件和状态迁移关系，是 TCP 协议实现的核心逻辑之一。基于当前的状态，如果收到 SYN，则会转到 SYN-RECEIVED 状态。

![](/data/image/network/nmap/image-8.png)

{{< hint info >}}
`TIME-WAIT` 的作用：
1. 在一段时间内保留该连接的状态。若本端发送给对方的最后一个 ACK 丢失，对方会重传 FIN，这时本端仍可重新发送 ACK，从而保证连接能够可靠结束。
2. 让网络中属于旧连接的延迟报文有足够时间过期消失，避免这些旧报文被后续同一四元组的新连接误认为是有效数据。

`TIME-WAIT` 的持续时间通常为 `2MSL`（`MSL`，Maximum Segment Lifetime，表示报文在网络中的最大生存时间）。之所以需要等待 `2MSL`，是因为本端发送的最后一个 `ACK` 最长可能需要 1 个 `MSL` 才能到达对方；如果该 `ACK` 丢失，对方重传 `FIN` 返回本端也最多还需要 1 个 `MSL`。因此，本端需要至少保留连接状态 `2MSL`，这样在对方未收到最后 `ACK` 而重传 `FIN` 时，仍然能够重新发送 `ACK`，从而保证连接可靠关闭。
{{< /hint >}}

---

## -sT

在本机进行扫描，扫描结束后**如果使用`FIN/ACK`四次挥手关闭**，则主动关闭连接的一方会进入 TIME-WAIT 状态，必须等待 2MSL （一般 60–120 秒）时间才会转为 CLOSED。这会导致：

- 扫描大量端口时，本机 netstat 会出现大量 TIME-WAIT
- 占用临时端口范围（ephemeral port），可能导致“端口耗尽”
- 内核需要维护这些 TCB，占用内存

`nmap`的`-sT` 调用内核的系统调用 `connect()` 建立连接，然后用户态通过 `setsockopt()` 配置 `SO_LINGER`进行`close()`，触发了异常的 **RST 关闭**，根本不会进入 `TIME-WAIT`状态。

```c
struct linger so;
so.l_onoff = 1;
so.l_linger = 0;
setsockopt(sock, SOL_SOCKET, SO_LINGER, &so, sizeof(so));
close(sock);
```

![](/data/image/network/nmap/image-9.png)

所以，我们查询本机的TCP连接状态，只能短暂的查询到一些连接，我们可以使用以下命令进行观测：

```bash
# 临时关闭 TIME-WAIT 复用，避免影响
sudo sysctl -w net.ipv4.tcp_tw_reuse=0

# 一个终端进行扫描
nmap -sT 67.xx.xx.226

# 另一个终端
watch -n 0.2 "ss -ant | grep 67.xx.xx.226"
```

![](/data/image/network/nmap/image-10.png)

![](/data/image/network/nmap/image-11.png)


## -sS

而使用 -sS（半开放扫描）时，发出的包全部由`nmap`在用户态自行构造，并不调用内核提供的系统调用。本机在接收到目标返回的 SYN+ACK 后，立即发送一个 RST 报文 来中止连接建立过程。这个 RST 会促使目标主机终止处于 SYN_RECEIVED 状态的半连接，从而释放对应的 TCB 资源，避免进入 ESTABLISHED 状态。

```bash
# 一个终端进行扫描
nmap -sS 67.230.188.226

# 另一个终端
watch -n 0.2 "ss -ant | grep 67.230.188.226"
```

![](/data/image/network/nmap/image-12.png)

我们查询本机的TCP连接状态，无法查到任何连接信息

![](/data/image/network/nmap/image-13.png)

> TCB 资源指的是 Transmission Control Block（传输控制块） 以及与它关联的各种内核资源。TCP 协议栈中每一个活动的连接都会分配一个 TCB，用来保存该连接的所有状态信息与运行所需的资源。它记录了一个 TCP 连接的四元组（源 IP、源端口、目标 IP、目标端口）以及连接生命周期中的各种状态信息。TCB 是 TCP 协议栈的核心数据结构之一。