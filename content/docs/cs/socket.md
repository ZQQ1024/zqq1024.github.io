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
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF               NODE NAME
...
server  2970  zqq    4u  IPv4  17770      0t0               TCP localhost:http-alt->localhost:47686 (ESTABLISHED)
...

lsof -p 2976
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF               NODE NAME
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

fork版本

```C
// server_fork.c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <signal.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <unistd.h>

int main() {
    signal(SIGCHLD, SIG_IGN); // 简化：自动回收子进程，避免僵尸，父进程不关心 SIGCHLD，内核直接回收

    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(8080);

    bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
    listen(lfd, 128);

    printf("listening on 0.0.0.0:%d\n", 8080);

    for (;;) {
        int cfd = accept(lfd, 0, 0);
        if (cfd < 0) continue;

        if (fork() == 0) {       // child
            close(lfd);          // 子进程不需要监听 fd

            char buf[1024];
            int n = read(cfd, buf, sizeof(buf));
            if (n > 0) write(cfd, buf, n);

            sleep(30);           // 方便观察 ESTABLISHED
            close(cfd);
            _exit(0);
        }

        close(cfd);              // 父进程不处理这个连接
    }
}
```

```bash
netstat -apn | grep 8080
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      744/./server_fork
tcp        0      0 127.0.0.1:8080          127.0.0.1:46764         ESTABLISHED 1905/./server_fork
tcp        0      0 127.0.0.1:47146         127.0.0.1:8080          ESTABLISHED 1889/./client
tcp        0      0 127.0.0.1:47162         127.0.0.1:8080          ESTABLISHED 1896/./client
tcp        0      0 127.0.0.1:8080          127.0.0.1:47146         ESTABLISHED 1890/./server_fork
tcp        0      0 127.0.0.1:46764         127.0.0.1:8080          ESTABLISHED 1904/./client
tcp        0      0 127.0.0.1:8080          127.0.0.1:47162         ESTABLISHED 1897/./server_fork

pstree -p 744
server_fork(744)─┬─server_fork(1890)
                 ├─server_fork(1897)
                 └─server_fork(1905)

lsof -p 744
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF               NODE NAME
...
server_fo 744  zqq    3u  IPv4   2951      0t0                TCP *:http-alt (LISTEN)

lsof -p 1890
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF               NODE NAME
...
server_fo 1890  zqq    4u  IPv4  14443      0t0                TCP localhost:http-alt->localhost:47146 (ESTABLISHED)
```

- fork()并不会立刻把内存整块拷贝一份，否则成本太高；而是创建一个新的 task_struct（调度实体），共享虚拟内存映射并配合COW，继承寄存器上下文：看起来像从 fork() 处继续执行，继承（复制）打开的文件描述符fd，继承当前工作目录、根目录、环境变量、用户/组 ID、权限等
- 当fork()返回时，内核会给父进程和子进程分别设置“返回值寄存器”如`x86-64：RAX`，`父进程：RAX = 子PID，子进程：RAX = 0`,父子两个进程从同一条“fork 返回后的指令”继续执行，但读到的返回值不同，于是走进不同的 `if/else` 分支
- 僵尸进程本质是子进程已经退出“死了”，但父进程还没来“收尸”；子进程调用 exit()结束，内核不会立刻把它彻底删除，会保留PID、退出码等信息，状态标记为`EXIT_ZOMBIE`；父进程有权知道子进程是怎么退出的，需要由父进程确认，可以通过`wait()/waitpid()`获得子进程信息
- fork()常和exec()搭配使用，fork用于得到一个子进程，exec通过把当前程序的代码、栈、堆全部替换为另外一个程序，而fd、环境变量等其他信息可以看情况继承

```
gcc server_pthread.c -o server_pthread -pthread
```

```C
// server_pthread.c
#include <arpa/inet.h>
#include <netinet/in.h>
#include <pthread.h>
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>
#include <stdlib.h>

static void* worker(void *arg) {
    int cfd = *(int*)arg;
    free(arg);

    char buf[1024];
    int n = read(cfd, buf, sizeof(buf));
    if (n > 0) write(cfd, buf, n);

    sleep(30);          // 方便观察 ESTABLISHED
    close(cfd);
    return NULL;
}

int main() {
    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(8080);

    bind(lfd, (struct sockaddr*)&addr, sizeof(addr));
    listen(lfd, 128);

    printf("listening on 0.0.0.0:%d\n", 8080);

    for (;;) {
        int cfd = accept(lfd, 0, 0);
        if (cfd < 0) continue;

        int *p = malloc(sizeof(int));
        *p = cfd;

        pthread_t tid;
        if (pthread_create(&tid, NULL, worker, p) == 0) {
            pthread_detach(tid);     // 不 join，线程退出自动回收资源
        } else {
            // 创建失败就同步处理/关闭
            free(p);
            close(cfd);
        }
    }
}
```

```bash
pstree -p 672
server_pthread(672)─┬─{server_pthread}(1352)
                    ├─{server_pthread}(1359)
                    └─{server_pthread}(1363)

netstat -apn | grep 8080                                                                       
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      672/./server_pthrea
tcp        0      0 127.0.0.1:8080          127.0.0.1:46372         ESTABLISHED 672/./server_pthrea
tcp        0      0 127.0.0.1:8080          127.0.0.1:46360         ESTABLISHED 672/./server_pthrea
tcp        0      0 127.0.0.1:8080          127.0.0.1:48428         ESTABLISHED 672/./server_pthrea
tcp        0      0 127.0.0.1:46360         127.0.0.1:8080          ESTABLISHED 1358/./client
tcp        0      0 127.0.0.1:46372         127.0.0.1:8080          ESTABLISHED 1362/./client
tcp        0      0 127.0.0.1:48428         127.0.0.1:8080          ESTABLISHED 1351/./client

lsof -p 672                                                                                    
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF               NODE NAME
...
server_pt 672  zqq    0u   CHR  136,0      0t0                  3 /dev/pts/0
server_pt 672  zqq    1u   CHR  136,0      0t0                  3 /dev/pts/0
server_pt 672  zqq    2u   CHR  136,0      0t0                  3 /dev/pts/0
server_pt 672  zqq    3u  IPv4   9529      0t0                TCP *:http-alt (LISTEN)
server_pt 672  zqq    4u  IPv4   9604      0t0                TCP localhost:http-alt->localhost:46360 (ESTABLISHED)
server_pt 672  zqq    5u  IPv4   9607      0t0                TCP localhost:http-alt->localhost:46372 (ESTABLISHED)
server_pt 672  zqq    7u  IPv4   9571      0t0                TCP localhost:http-alt->localhost:48428 (ESTABLISHED)

lsof -p 1352 # 返回为空
ps aux | grep 1352 # 

ps -T -p 672
PID    SPID TTY          TIME CMD
672     672 pts/0    00:00:00 server_pthread
672    1351 pts/0    00:00:00 server_pthread
672    1358 pts/0    00:00:00 server_pthread
672    1362 pts/0    00:00:00 server_pthread
```

- 内核唯一的调度实体是`task_struct`，并不区分进程、线程，内核里全是task，每个task都有一个`pid`或者叫做`tid`
- “线程”是用户态的概念，为了让这其成立，内核引入了`tgid`(Thread Group ID)这一概念，用于标识一组共享资源（地址空间、fd 表等）的同一个线程组，线程组leader的pid 等于线程的tgid
- 用户态一些工具的输出，可能会导致一些错误的理解，但是这是具体用户态工具对内核数据的解释行为：
    - `getpid()`返回的实质是`task_struct.tgid`
    - `gettid()`返回的实质是内核task的`task_struct.pid`
    - `ps aux`按进程视角展示，一个线程组合并只展示一条，展示的那条记录对应 thread group leader
    - `lsof -p`也是按`tgid`展示的，指定子线程`pid`结果为空，子线程共享进程的fd表，从fd的角度，根本分不出是哪个线程在用，并不能把线程当作独立的fd拥有者
    - `pstree`将tgid相同的task视为进程，如上面输出中的`672`，将相同tgid但非leader的task视为线程，如`1351/1358/1362`，并用`{}`标识
- pthread是POSIX 线程标准库（POSIX Threads），是 POSIX 标准的一部分，多线程编程的基础，本质是通过调用 `clone()`，创建一个新的 task，指定一些flags，让它共享一堆资源，如`CLONE_FILES`共享 fd 表，`CLONE_THREAD`同一线程组（TGID 相同）。
- `clone()` 是 Linux 的“底层原语”，`fork()` 是在其之上封装的固定用法，`clone 不带 CLONE_THREAD ≈ fork`，`fork()`的语义是创建新的进程会产生新的tgid。
- fd 的设计思想就是对资源的引用，close(fd) 是释放引用，是否释放资源取决于引用是否归零。unlink 文件后，正在打开的进程还能继续读写：unlink("a.txt") 做的事情是把目录里 a.txt → inode 这条映射删除，再也无法通过文件名打开它，open() 之后，进程并不是“拿着文件名在读写”，而是 fd → file struct → inode，读写行为完全正常，lsof 能看到 (deleted) 但磁盘空间还没释放，只有引用计数（link count）变为0，才会真正释放 inode 和数据块
- 多线程编程时经常看到`join`，`join`的作用是什么？线程在退出后，会留下线程的返回值和其他一些资源，需要由另一个线程拿到`pthread_t tid`调用 `pthread_join(tid, NULL)` 回收（一般是主线程但也可是其他子线程），否则就产生了类似僵尸进程一样的问题，也可以使用`pthread_detach(tid)`将线程标记为不可join，线程结束后自动回收其资源；`join`并不是指将多个线程合并成一条，而是在某一时间节点等待线程执行完，一般的使用场景是`主线程split了多个线程，并发执行，然后主线程join汇合`
- POSIX全称是Portable Operating System Interface，指的是“用户态的接口规范”，不是内核提供的系统调用集合，规定“用户程序应该看到什么接口、这些接口语义是什么”。libc（C standard library）为 C 程序提供一组基础函数的库的统称，libc = ISO C 标准(printf,malloc) + POSIX 标准(fork,pthread_*) + 平台扩展(BSD/macOS) 的实现集合，实现有`glibc(GNU C Library)`，`musl(Alpine Linux)`等


## I/O multiplexing

分析上面fork/pthread例子的本质，我们发现：
- 一个连接对应一个执行实体（进程/线程），如果连接数是 N，则进程/线程数也是 N，资源线性增长，
- 进程/线程上下文需要切换
- 线程/进程大部分时间在等待外部I/O，read可读，write可写之类的，其他什么都不干，但资源一直占着

可以得出上面的通讯模型性能高不了，同时带来一个问题：能不能用少量线程，同时盯住大量fd，只在真正需要时才去处理？

这就是 I/O 多路复用(I/O multiplexing)

I/O 多路复用是一种 I/O 等待模型，它通过在单个或少量线程中统一等待多个文件描述符的 I/O 就绪事件，避免为每个 I/O 对象创建独立的阻塞执行实体，提高并发能力和资源利用率。

I/O 多路复用监听的不是“连接”这一高层语义，而是 fd 级别的 I/O 事件，即 fd 当前是否具备可读、可写或异常等条件。通过这一机制，可以将网络通信、管道、设备文件等统一抽象为 I/O 事件进行处理。


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

select 和 poll 本质一样：都是“线性扫描 fd 的就绪状态”。

select(maxfd + 1, &rset, &wset, &eset, NULL); 
“我帮你检查 0 ~ maxfd 之间所有 fd，看位图里哪些被置位了。”即使关心部分fd，也要全部扫描。fd分布比较稀疏，则性能损失较大。
FD_SETSIZE 通常是1024

用 struct pollfd[]（数组），不需要 maxfd，无数量的硬限制
poll(pfds, nfds, -1); “我只检查你数组里列出的这些 fd。”

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

### epoll

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