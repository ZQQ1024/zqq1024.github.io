---
weight: 3
bookCollapseSection: false
title: "socket编程"
---

关于socket知识点过一段时间就会忘，这里通过几个代码示例用于支撑一些结论性，方便后续回忆

```C
// server.c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

int main() {
    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(8080);

    bind(lfd, (struct sockaddr*)&addr, sizeof(addr));

    // passive open
    // lfd 是 listening socket 监听套接字，只能等连接，不能读写用户数据
    // 对应状态是 LISTEN
    listen(lfd, 1);

    
    printf("listening on 0.0.0.0:%d\n", 8080);
    
    // connected socket 已连接套接字，真正用来 read/write 的 fd
    // 对应状态是 ESTABLISHED
    int cfd = accept(lfd, NULL, NULL);

    char buf[1024];
    int n = read(cfd, buf, sizeof(buf));
    write(cfd, buf, n);

    // 保持连接一段时间，方便netstat观察
    sleep(30);

    close(cfd);
    close(lfd);
    return 0;
}
```

```C
// client.c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

int main() {
    int fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);

    // active open
    connect(fd, (struct sockaddr*)&addr, sizeof(addr));

    char *msg = "hi\n";
    write(fd, msg, strlen(msg));

    char buf[1024];
    int n = read(fd, buf, sizeof(buf) - 1);
    buf[n] = 0;
    printf("%s", buf);

    // 保持连接一段时间，方便netstat观察
    sleep(30);

    close(fd);
    return 0;
}

```

编译后运行，
```bash
./server                                                               
listening on 0.0.0.0:8080

./client                                                             
hi

netstat -apn | grep 8080                                               
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      2970/./server
tcp        0      0 127.0.0.1:8080          127.0.0.1:47686         ESTABLISHED 2970/./server
tcp        0      0 127.0.0.1:47686         127.0.0.1:8080          ESTABLISHED 2976/./client

lsof -p 2970
...
server  2970  zqq    4u  IPv4  17770      0t0               TCP localhost:http-alt->localhost:47686 (ESTABLISHED)
...

lsof -p 2976
...
client  2976  zqq    3u  IPv4  17771      0t0               TCP localhost:47686->localhost:http-alt (ESTABLISHED)
...

# 等待一会后

netstat -apn | grep 8080                                               
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 127.0.0.1:8080          127.0.0.1:47686         TIME_WAIT   -
tcp        0      0 127.0.0.1:47686         127.0.0.1:8080          TIME_WAIT   -
```

- 在Linux Everything is a file的背景下，`socket()`和`open()`都会得到一个文件描述符fd，之后就可以用很多相同的 I/O 接口去操作它，来实现文件读写，报文收发等

- socket 的本意是“插座 / 接口”，语义是提供一个通用的通信端点接口，代表通信的一端；用于统一不同通信域与协议的使用方式，并在用户态屏蔽底层协议栈的实现细节
    - 代表通信的一端，可被 bind / connect / listen / accept，可读可写
    - read/write/send/recv，是围绕这个端点展开的可操作集合
    - 统一通信域，本机：AF_UNIX，v4/v6网络：AF_INET / AF_INET6 等
    - 屏蔽了TCP 状态机，拥塞控制等实现细节，但并没有屏蔽TCP 是流，UDP 是报文，阻塞 / 非阻塞这些使用时的语义差异

- 可以在socket对应的fd设置`阻塞 / 非阻塞`标志位，本质是当事情暂时做不了时，线程该怎么办，是挂起还是立即返回报错。阻塞（blocking）典型表现是以下操作时，线程会被挂起：
    - read()时没数据
    - write()时发送缓冲区满
    - accept()时没有新连接
- `阻塞 / 非阻塞`是fd的属性，不是socket的。在 socket、pipe、FIFO、设备文件上也可以涉及`阻塞 / 非阻塞`


## 多client

fork/pthread

## I/O multiplexing

### select

### poll

### epoll