---
weight: 3
bookCollapseSection: false
title: "I/O Multiplexing"
---

## I/O Multiplexing

分析上面fork/pthread例子的本质，我们发现：
- 一个连接对应一个执行实体（进程/线程），如果连接数是 N，则进程/线程数也是 N，资源线性增长，
- 进程/线程上下文需要切换
- 线程/进程大部分时间在等待外部I/O，read可读，write可写之类的，其他什么都不干，但资源一直占着

可以得出上面的通讯模型性能高不了，同时带来一个问题：**能不能用少量线程，同时盯住大量fd，只在真正需要时才去处理？** 这就是 I/O 多路复用(I/O multiplexing)

I/O 多路复用是一种 I/O 等待模型，它通过在单个或少量线程中统一等待多个文件描述符的 I/O 就绪事件，避免为每个 I/O 对象创建独立的阻塞执行实体，提高并发能力和资源利用率。

I/O 多路复用监听的不是“连接”这一高层语义，而是 fd 级别的 I/O 事件，即 fd 当前是否具备可读、可写或异常等条件。通过这一机制，可以将网络通信、管道、设备文件等统一抽象为 I/O 事件进行处理

### select

基本的select+non-blocking的读写例子

```C
// server_select_nb.c
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <sys/select.h>
#include <sys/socket.h>
#include <unistd.h>

static void set_nonblock(int fd) {
    int f = fcntl(fd, F_GETFL, 0);
    // 原有flag上添加非阻塞
    fcntl(fd, F_SETFL, f | O_NONBLOCK);
}

int main() {
    signal(SIGPIPE, SIG_IGN); // 对端关闭时 write 不要把进程打死

    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    // int yes = 1;
    // setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));
    
    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(8080);

    bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
    listen(lfd, 128);

    printf("listening on 0.0.0.0:%d\n", 8080);

    fd_set all, rset; 
    FD_ZERO(&all);

    FD_SET(lfd, &all);
    int maxfd = lfd;

    for (;;) {
        rset = all; // select 会修改传入集合，所以每轮都要复制一份
        // 把 &rset 作为 readfds可读就绪事件 传给了 select，writefds可写就绪事件，exceptfds异常就绪事件
        if (select(maxfd + 1, &rset, NULL, NULL, NULL) <= 0) continue; //阻塞等待

        if (FD_ISSET(lfd, &rset)) { // 如果监听 fd 可读：说明 accept 队列里有新连接
            int cfd = accept(lfd, NULL, NULL);
            if (cfd >= 0) {
                set_nonblock(cfd);        // 非阻塞写（也会让 read 变非阻塞）
                FD_SET(cfd, &all);
                if (cfd > maxfd) maxfd = cfd; // 更新 maxfd，保证 select 监控范围覆盖到所有连接的fd
            }
        }

        for (int fd = lfd + 1; fd <= maxfd; fd++) {
            if (!FD_ISSET(fd, &rset)) continue; // 如果该 fd 本轮不“可读”，跳过
        
            char buf[1024];
            ssize_t n = read(fd, buf, sizeof(buf));
            if (n > 0) {
                // 非阻塞写：处理部分写，遇到 EAGAIN/EWOULDBLOCK 就“先丢掉剩余”（演示用）
                ssize_t off = 0;
                while (off < n) {
                    ssize_t w = write(fd, buf + off, (size_t)(n - off));
                    if (w > 0) { off += w; continue; }
                    if (w < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) break;
                    if (w < 0 && errno == EINTR) continue;
                    // 其他错误：关闭
                    close(fd);
                    FD_CLR(fd, &all);
                    break;
                }
            } else if (n == 0) {
                close(fd);
                FD_CLR(fd, &all);
            } else {
                if (errno == EAGAIN || errno == EWOULDBLOCK || errno == EINTR) continue;
                close(fd);
                FD_CLR(fd, &all);
            }
        }
    }
}
```

```bash
netstat -apn | grep 8080                                 
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      695/./server_select
tcp        0      0 127.0.0.1:8080          127.0.0.1:51352         ESTABLISHED 695/./server_select
tcp        0      0 127.0.0.1:8080          127.0.0.1:51362         ESTABLISHED 695/./server_select
tcp        0      0 127.0.0.1:51928         127.0.0.1:8080          ESTABLISHED 895/./client
tcp        0      0 127.0.0.1:8080          127.0.0.1:51928         ESTABLISHED 695/./server_select
tcp        0      0 127.0.0.1:51362         127.0.0.1:8080          ESTABLISHED 912/./client
tcp        0      0 127.0.0.1:51352         127.0.0.1:8080          ESTABLISHED 905/./client

lsof -p 695                                              
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF              NODE NAME
...
server_se 695  zqq    0u   CHR  136,0      0t0                 3 /dev/pts/0
server_se 695  zqq    1u   CHR  136,0      0t0                 3 /dev/pts/0
server_se 695  zqq    2u   CHR  136,0      0t0                 3 /dev/pts/0
server_se 695  zqq    3u  IPv4   4551      0t0               TCP *:http-alt (LISTEN)
server_se 695  zqq    4u  IPv4   4588      0t0               TCP localhost:http-alt->localhost:51928 (ESTABLISHED)
server_se 695  zqq    5u  IPv4   4589      0t0               TCP localhost:http-alt->localhost:51352 (ESTABLISHED)
server_se 695  zqq    6u  IPv4   4591      0t0               TCP localhost:http-alt->localhost:51362 (ESTABLISHED)

netstat -apn | grep 8080
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      695/./server_select
tcp        0      0 127.0.0.1:51928         127.0.0.1:8080          TIME_WAIT   -
tcp        0      0 127.0.0.1:51362         127.0.0.1:8080          TIME_WAIT   -
tcp        0      0 127.0.0.1:51352         127.0.0.1:8080          TIME_WAIT   -
```

- 代码中的buf并不是我们说的“缓冲区”，他只是用户代码层面用于存放read()到的数据或者准备write()的数据
- 缓冲区是“每个 socket 的概念”，接收缓冲区用于存储已经从网络接收到内核态但并没有被用户态程序取走的数据，发送缓冲区用于存储已经从用户态接收到内核态但是并没有发送到网络中
- TCP是面向数据流的，并没有明显的数据包的边界，write(100 bytes)和write(20 bytes) 了 5 次对端接收来看没有区别，read()/write()能实际读多少/写多少是没法确定的
- non-blocking/blocking在socket上体现就是，read()没数据就等，write()发送缓冲区满就等，accept()没有新连接就等
- 上面的代码例子使用non-blocking，因为程序真题就一个线程，如果 blocking 整个代码就卡住了影响其他请求的处理

### poll

```C
// server_poll_nb.c
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <poll.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

static void set_nonblock(int fd) {
    int f = fcntl(fd, F_GETFL, 0);
    if (f >= 0) (void)fcntl(fd, F_SETFL, f | O_NONBLOCK);
}

static void close_and_remove(struct pollfd *pfds, int *nfds, int idx) {
    close(pfds[idx].fd);
    // swap with last
    pfds[idx] = pfds[*nfds - 1];
    (*nfds)--;
}

int main() {
    signal(SIGPIPE, SIG_IGN);

    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if (lfd < 0) { perror("socket"); return 1; }
    set_nonblock(lfd);

    // int yes = 1;
    // if (setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes)) < 0) perror("setsockopt");

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(8080);

    if (bind(lfd, (struct sockaddr *)&addr, sizeof(addr)) < 0) { perror("bind"); return 1; }
    if (listen(lfd, 128) < 0) { perror("listen"); return 1; }

    printf("listening on 0.0.0.0:%d\n", 8080);

    // pollfd 动态数组，可扩容
    int cap = 1024;
    struct pollfd *pfds = (struct pollfd *)calloc((size_t)cap, sizeof(struct pollfd));
    if (!pfds) { perror("calloc"); return 1; }

    int nfds = 0;
    pfds[nfds].fd = lfd;
    pfds[nfds].events = POLLIN; // 有新连接可 accept
    nfds++;

    for (;;) {
        // 阻塞等待
        int ready = poll(pfds, (nfds_t)nfds, -1);
        if (ready < 0) {
            if (errno == EINTR) continue;
            perror("poll");
            break;
        }

        // 1) 先处理监听 fd：accept 新连接
        if (pfds[0].revents & POLLIN) {
            for (;;) {
                int cfd = accept(lfd, NULL, NULL);
                if (cfd < 0) {
                    if (errno == EAGAIN || errno == EWOULDBLOCK) break; // accept 队列空了
                    if (errno == EINTR) continue;
                    perror("accept");
                    break;
                }

                set_nonblock(cfd);

                if (nfds == cap) {
                    cap *= 2; // 容量翻倍
                    // realloc 可能移动内存地址，返回新指针
                    struct pollfd *np = (struct pollfd *)realloc(pfds, (size_t)cap * sizeof(struct pollfd));
                    if (!np) { perror("realloc"); close(cfd); break; }
                    pfds = np;
                }

                pfds[nfds].fd = cfd;
                pfds[nfds].events = POLLIN; // echo demo：只关心可读
                pfds[nfds].revents = 0;
                nfds++;
            }
        }

        // 2) 处理所有客户端 fd
        // 注意：从 1 开始，因为 0 是 lfd
        for (int i = 1; i < nfds; ) {
            short re = pfds[i].revents;
            pfds[i].revents = 0;

            // 对端挂了/错误
            if (re & (POLLERR | POLLHUP | POLLNVAL)) {
                close_and_remove(pfds, &nfds, i);
                continue; // i 不自增，因为换进来一个新元素
            }

            if (!(re & POLLIN)) { // 本轮不可读
                i++;
                continue;
            }

            char buf[1024];
            ssize_t n = read(pfds[i].fd, buf, sizeof(buf));
            if (n > 0) {
                ssize_t off = 0;
                while (off < n) {
                    ssize_t w = write(pfds[i].fd, buf + off, (size_t)(n - off));
                    if (w > 0) { off += w; continue; }
                    if (w < 0 && errno == EINTR) continue;
                    if (w < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) break; // 演示：剩余直接丢
                    // 其他错误：关闭连接
                    close_and_remove(pfds, &nfds, i);
                    goto next_i; // i 不自增
                }
                i++;
            } else if (n == 0) {
                close_and_remove(pfds, &nfds, i);
            } else {
                if (errno == EINTR) { i++; }
                else if (errno == EAGAIN || errno == EWOULDBLOCK) { i++; }
                else { close_and_remove(pfds, &nfds, i); }
            }

        next_i:
            ;
        }
    }

    // cleanup
    for (int i = 0; i < nfds; i++) close(pfds[i].fd);
    free(pfds);
    return 0;
}
```

bash输出基本同上
- select 和 poll 本质一样：都是“线性扫描 fd 的就绪状态”。`select(maxfd + 1, &rset, &wset, &eset, NULL); `
“检查 0 ~ maxfd 之间所有 fd，看位图里哪些被置位了。”即使只关心部分fd，也要全部扫描。fd分布比较稀疏，则性能损失较大。
FD_SETSIZE 通常是1024
- select 用 `struct pollfd[]`（数组），不需要 maxfd，无数量的硬限制。`poll(pfds, nfds, -1);` “只检查你数组里列出的这些 fd。”

### epoll

基于事件的poll

```C
// server_epoll_nb.c
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

static void set_nonblock(int fd) {
    int f = fcntl(fd, F_GETFL, 0);
    if (f >= 0) fcntl(fd, F_SETFL, f | O_NONBLOCK);
}

static void close_fd(int fd) {
    if (fd >= 0) close(fd);
}

int main() {
    signal(SIGPIPE, SIG_IGN);

    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    set_nonblock(lfd);

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(8080);

    bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    listen(lfd, 128);

    printf("listening on 0.0.0.0:8080\n");

    int ep = epoll_create1(0);

    struct epoll_event ev;
    memset(&ev, 0, sizeof(ev));
    ev.events = EPOLLIN;
    ev.data.fd = lfd;
    epoll_ctl(ep, EPOLL_CTL_ADD, lfd, &ev);

    struct epoll_event events[1024];

    for (;;) {
        int nready = epoll_wait(ep, events, 1024, -1);

        for (int i = 0; i < nready; i++) {
            int fd = events[i].data.fd;          // 取出事件对应的 fd
            uint32_t re = events[i].events;      // 取出就绪事件标志位

            // EPOLLERR: 异常错误
            // EPOLLHUP: 挂断
            // EPOLLRDHUP: 对端关闭写端（半关闭）或断开
            if (re & (EPOLLERR | EPOLLHUP | EPOLLRDHUP)) {
                epoll_ctl(ep, EPOLL_CTL_DEL, fd, NULL);
                close_fd(fd);
                continue;
            }

            if (fd == lfd) {
                for (;;) {
                    int cfd = accept(lfd, NULL, NULL);
                    if (cfd < 0) break;

                    set_nonblock(cfd);

                    // 把客户端 fd 加入 epoll，关注可读 + 对端半关闭
                    struct epoll_event cev;
                    memset(&cev, 0, sizeof(cev));
                    cev.events = EPOLLIN | EPOLLRDHUP;
                    cev.data.fd = cfd;
                    epoll_ctl(ep, EPOLL_CTL_ADD, cfd, &cev);
                }
                continue;                              // 监听 fd 处理完就进下一条事件
            }

            if (re & EPOLLIN) {
                char buf[1024];

                for (;;) {
                    ssize_t rn = read(fd, buf, sizeof(buf));
                    if (rn <= 0) {
                        // rn==0：对端正常关闭
                        // rn<0 且不是 EAGAIN：读出错
                        if (rn == 0 || (rn < 0 && errno != EAGAIN && errno != EWOULDBLOCK)) {
                            epoll_ctl(ep, EPOLL_CTL_DEL, fd, NULL);
                            close_fd(fd);
                        }
                        break; // rn<0 且 EAGAIN：读空了，结束本次读循环
                    }

                    // 写回（echo）写不完就丢（不做发送缓冲）
                    ssize_t off = 0;
                    while (off < rn) {
                        ssize_t wn = write(fd, buf + off, (size_t)(rn - off));
                        if (wn > 0) { off += wn; continue; }
                        break; // EAGAIN/EINTR/其他错误都直接停
                    }
                }
            }
        }
    }
}
```

bash输出和上面没什么区别

poll 是“每次都把一堆 fd 拿去问一遍”，epoll 是“先登记规则，内核在状态变化时通知你”。
本质是等待模型从“轮询”变成了“事件订阅”。