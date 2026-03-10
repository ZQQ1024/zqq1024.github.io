---
weight: 2
bookToc: true
title: "channel"
---

## 背景

`channel` 底层其实是一个利用互斥锁（mutex）+ 队列来协调 `goroutine` 间数据通信的数据结构。符合**不要通过共享内存来通信，而要通过通信来共享内存**的思想。

下面会从 `channel` 的几个特性展开介绍 `channel` 的一些原理机制：
- `channel` 是 `goroutine-safe` 的，`goroutine` 之间可以互相传值，`FIFO`风格
- `channel` 可以引起 `goroutine` 的 `block` 和 `unblock`

---

## hchan

`channel` 本质结构是 `hchan` 代码在

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

TODO 字段含义

## make chan

`ch := make(chan Task, 3)` / `ch := make(chan Task)` 对应 `runtime.makechan` 代码在

[https://github.com/golang/go/blob/master/src/runtime/chan.go#L75](https://github.com/golang/go/blob/master/src/runtime/chan.go#L75)

`func makechan(t *chantype, size int) *hchan {...}` 在 `heap`上分配了一个`hchan`，同时返回的是 `*hchan`，所以多个 `goroutine` 操作同一个 `channel` 时，实际是在共享同一个 `*hchan` 指针，指向了同一个`hchan`，不会复制 `hchan` 结构体

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

所以本质是一个加锁的队列


> https://github.com/gophercon/2017-talks/blob/master/KavyaJoshi-UnderstandingChannels/Kavya%20Joshi%20-%20Understanding%20Channels.pdf  
https://www.classcentral.com/course/youtube-gophercon-2017-kavya-joshi-understanding-channels-235585


