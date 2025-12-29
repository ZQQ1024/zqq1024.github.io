---
weight: 2
bookCollapseSection: false
title: "多客户端"
---

## Multi

分为基于fork/pthread的版本

### fork


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

### pthread

编译时需要指定`pthread`库
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
- POSIX全称是Portable Operating System Interface，指的是“用户态的接口规范”，不是内核提供的系统调用集合，规定“用户程序应该看到什么接口、这些接口语义是什么”。libc（C standard library）为 C 程序提供一组基础函数的库的统称，libc = ISO C 标准(printf,malloc) + POSIX 标准(fork,pthread_*) + 平台扩展(BSD/macOS) 的实现集合；实现有`glibc(GNU C Library)`，`musl(Alpine Linux)`等