---
weight: 2
bookToc: true
title: "channel"
---

## 背景

`channel` 底层其实是一个利用互斥锁（mutex）+ 队列来协调 `goroutine` 间数据通信的数据结构。符合 **不要通过共享内存来通信，而要通过通信来共享内存** 的思想。

下面会从 `channel` 的几个特性展开介绍 `channel` 的一些原理机制：
- `channel` 是 `goroutine-safe` 的，`goroutine` 之间可以互相传值，`FIFO`风格
- `channel` 可以引起 `goroutine` 的 `block` 和 `unblock`

---

## hchan & make chan

在 `Go runtime` 中 `channel` 的底层数据结构是 `hchan` ，代码在

[https://github.com/golang/go/blob/master/src/runtime/chan.go#L34](https://github.com/golang/go/blob/master/src/runtime/chan.go#L34)

```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	timer    *timer // timer feeding this chan
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters
	bubble   *synctestBubble

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

主要字段含义：

| 字段 | 类型 | 含义 |
|------|------|------|
| `buf` | `unsafe.Pointer` | 指向**循环队列**数组的指针，**仅用于有缓冲 channel** |
| `dataqsiz` | `uint` | 循环队列的容量，即 `make(chan T, N)` 中的 N |
| `qcount` | `uint` | 队列中当前存放的元素数量 |
| `elemsize` | `uint16` | 每个元素的字节大小 |
| `elemtype` | `*_type` | 元素的类型信息，用于内存拷贝和 GC |
| `sendx` | `uint` | 下一次写入的位置 |
| `recvx` | `uint` | 下一次读取的位置 |
| `recvq` | `waitq` |  `channel` 空而阻塞的 `goroutine` 链表（等待接收） |
| `sendq` | `waitq` |  `channel` 满而阻塞的 `goroutine` 链表（等待发送） |
| `closed` | `uint32` |  0 = 开启，1 = 已关闭；close(ch) 后置 1 |
| `lock` | `waitq` |  互斥锁，保护上述**所有字段**的并发访问 |


![alt text](/data/image/language/golang/channel/image.png)

---

`ch := make(chan Task, 3)` / `ch := make(chan Task)` 对应 `Go runtime` 中的 `runtime.makechan` ，代码在

[https://github.com/golang/go/blob/master/src/runtime/chan.go#L75](https://github.com/golang/go/blob/master/src/runtime/chan.go#L75)

`func makechan(t *chantype, size int) *hchan {...}` 在 `heap`上分配了一个`hchan`，同时返回的是 `*hchan`，所以多个 `goroutine` 操作同一个 `channel` 时，实际是在共享同一个 `*hchan` 指针，指向了同一个`hchan`，不会复制 `hchan` 结构体

![alt text](/data/image/language/golang/channel/image-1.png)

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 1)  // ch 本质是 *hchan 指针，指向堆上一块内存

    go func() {
        ch <- 42            // 闭包捕获
    }()

    go func(ch chan int) {
        v := <-ch           // 值传递
        fmt.Println(v)
    }(ch)
}
```

## send & receive

对于 `channel`，本质是一个加锁的环形队列，所以是 `goroutine-safe` 的，`goroutine` 之间可以互相传值，`FIFO`风格

以下例子，通过 `send/receive` `channel` 实现跨 `goroutine` 的通讯

![alt text](/data/image/language/golang/channel/image-2.png)

![alt text](/data/image/language/golang/channel/image-3.png)

`G1` 获得锁，入队，`task0` 是通过内存 **copy** 到 `channel` 中的 `buf` 的，然后释放锁
![alt text](/data/image/language/golang/channel/image-4.png)

![alt text](/data/image/language/golang/channel/image-5.png)

![alt text](/data/image/language/golang/channel/image-6.png)

`G2` 获得锁，出队，`channel` 中的 `buf` 对应元素通过内存 **copy** 到 `task0`，然后释放锁

![alt text](/data/image/language/golang/channel/image-7.png)

![alt text](/data/image/language/golang/channel/image-8.png)

![alt text](/data/image/language/golang/channel/image-9.png)

![alt text](/data/image/language/golang/channel/image-10.png)

通过这个收发的例子，直观感受到了 **不要通过共享内存来通信，而要通过通信来共享内存** 的含义

![alt text](/data/image/language/golang/channel/image-11.png)

## block & unblock

> https://github.com/gophercon/2017-talks/blob/master/KavyaJoshi-UnderstandingChannels/Kavya%20Joshi%20-%20Understanding%20Channels.pdf  
https://www.classcentral.com/course/youtube-gophercon-2017-kavya-joshi-understanding-channels-235585


