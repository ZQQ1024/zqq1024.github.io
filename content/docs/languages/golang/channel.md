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

以下例子（针对有缓冲的 `channel`），通过 `send/receive` `channel` 实现跨 `goroutine` 的通讯

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

{{< hint info >}}
对于无缓冲的 `channel` buf 是 nil，`dataqsiz = 0`，`qcount` 永远为 0。数据传输时，完全绕过 buf，数据直接在 `goroutine` 之间传递，如 `receive first` 场景

G1: `v := <- ch` 阻塞等待，将 G1 封装成 `sudog`，`sudog.elem` 指向接收变量地址 `&v`，加入到 `ch.recvq` 队列中

G2: `ch <- v` 发现 `ch.recvq` 队列非空，`ch.recvq` 出队，直接调用 `typedmemmove` 把数据拷贝到 `sudog.elem`（即 G1 栈上的 `v`），**拷贝完成后**再调用 `goready` 唤醒 G1（顺序不能反，否则 G1 唤醒后栈可能发生收缩，导致 `sudog.elem` 成为野指针）

`send first` 同理，`sudog.elem` 指向发送值地址 `&x`，G2 阻塞；G1 到来后直接从 `sudog.elem` 读取数据，再唤醒 G2
{{< /hint >}}

## block & unblock

考虑以下 `send first`的场景，` ch <- task4`


**当 `channel` 满了，对应 send 的 `goroutine` 会暂停执行，并在 `channel` 被 `receive` 之后恢复执行**

![alt text](/data/image/language/golang/channel/image-12.png)

{{< hint info >}}
`goroutine` 是用户态线程，由 `Go runtime` 创建和管理，比 OS thread 轻量得多。

`runtime scheduler` 负责将 `M` 个 `goroutine` 调度到 `N` 个 OS thread 上执行，即 `M:N` 调度。

如下图，`g1 g2 g6` 等多个 `goroutine` 分时复用 OS `thread1`，`g3 g5 g4` 等复用 OS `thread2`，一个`goroutine`阻塞让出执行，就会去执行其他`goroutine`，同一个 goroutine（如 g1）也可以先后跑在不同的 OS thread 上。

![alt text](/data/image/language/golang/channel/image-13.png)

主要通过这种 `M`:`N` 调度模型将调度权从OS收回，不会让 OS thread 空转(starve)，且 `goutine`切换成本很低，以提高并发，详细会在GMP相关的文章展开进行介绍。

![alt text](/data/image/language/golang/channel/image-14.png)
{{< /hint >}}

---

### sudog

`sudog` 是 `pseudo-g`，代表一个等待的 `goroutine` 的代理对象。[runtime2 源码注释](https://github.com/golang/go/blob/master/src/runtime/runtime2.go#L394C1-L403C43) 如下：

```
// sudog (pseudo-g) represents a g in a wait list, such as for sending/receiving
// on a channel.
//
// sudog is necessary because the g ↔ synchronization object relation
// is many-to-many. A g can be on many wait lists, so there may be
// many sudogs for one g; and many gs may be waiting on the same
// synchronization object, so there may be many sudogs for one object.
//
// sudogs are allocated from a special pool. Use acquireSudog and
// releaseSudog to allocate and free them.
```

意思是 **G 和 同步对象（channel / mutex / semaphore）是多对多的关系**，举例解释：
```golang
// 一个 G 等待多个channel：
// 同一个 G，同时有 3 个 sudog 分散在 3 个 channel 队列里

select {
case ch1 <- v:   // sudog1 挂在 ch1.sendq
case ch2 <- v:   // sudog2 挂在 ch2.sendq  
case <-ch3:      // sudog3 挂在 ch3.recvq
}

// 多个 G 等待同一个 channel：一个同步对象对应多个 sudog
ch1.recvq → sudog(G1) → sudog(G2) → sudog(G3)
```

`sudog` 结构体包含 `next/prev` 指针，可以形成链表结构（`waitq` 是 `sudog` 组成的双向链表），通过 `sudog` 这一层代理实现了上面的举例

`sudog` 在下图中的主要作用是：G1 挂起后，`task4` 还在 G1 的栈上，`sudog.elem` 指向它，当 `receiver` 到来时直接从 `sudog.elem` 读取，不需要 G1 醒来参与，数据就被 `channel` 取走。同时，`receiver` 取完数据后，通过 `sudog.g` 找到 G1，调用 `goready(G1)` 将其唤醒。

![alt text](/data/image/language/golang/channel/image-16.png)

`sudog` 必须在 `gopark` 之前入队，否则 G1 睡着了，`receiver` 找不到它，数据也取不到。

### gopark

![alt text](/data/image/language/golang/channel/image-15.png)

[`gopark`](https://github.com/golang/go/blob/master/src/runtime/proc.go#L449) 是 `Go runtime` 中让当前 `goroutine` 挂起（阻塞） 的核心函数。

```golang
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, 
reason waitReason, traceReason traceBlockReason, traceskip int) {...}
```

当前**正在运行的 `goroutine` 主动调用 `gopark`**，调用后当前 G 状态从 `_Grunning` → `_Gwaiting`，从 M 上摘下，M 可以去执行其他 G。本质是一次主动让权（`cooperative yield`）。

### goready

当 G2 receive 之后，需要将 G1 唤醒，先通过 `sudog.elem` 将 `task4`入 `buf`，然后 G2 通过调用 `goready` 唤醒 G1

![alt text](/data/image/language/golang/channel/image-18.png)

![alt text](/data/image/language/golang/channel/image-17.png)

[`goready`](https://github.com/golang/go/blob/master/src/runtime/proc.go#L485) 是 `Go runtime` 中 `gopark` 的反操作，将一个 `_Gwaiting` 的 G 标记为 `_Grunnable`，放入运行队列等待调度。

```golang
func goready(gp *g, traceskip int) {
    systemstack(func() {
        ready(gp, traceskip, true)
    })
}
```

## 其他情况

### `receive first`

上面是`send first`的场景，考虑 `receive first` 即，`receive from a empty channel` 的场景

G2 执行`t := <- ch`阻塞，
![alt text](/data/image/language/golang/channel/image-19.png)

G1 执行`ch <- task`，G1 通过 `sudog.elem` 直接写 G2 的栈空间，不使用 `channel` 的 `buf` 避免再绕一下，以提升效率

![alt text](/data/image/language/golang/channel/image-20.png)

![alt text](/data/image/language/golang/channel/image-21.png)

### unbuffered channels

类似上面，不使用 `channel` 的 `buf`：
- `receive first` 情况：`sender` 直接写 `receiver` 栈地址
- `send first` 情况：`receiver` 直接通过 `sudog.elem` 获得数据 

### select

`select` 实现了多路 `channel` 的等待，哪个 channel 先就绪就走哪个 case，其他的丢弃。当多个 case 同时就绪时，Go runtime 随机选一个，避免饥饿。有 default 时，变为**非阻塞等待**，所有 channel 都没就绪直接走 default，**G 不会挂起**：

```golang
select {
	case ch1 <- v:
	case v := <-ch2:
	case v := <-ch3:
}
```

为了保证操作原子性，按**固定顺序**对所有 channel 加锁（固定顺序防止死锁）：
```
lock(ch1), lock(ch2), lock(ch3)
```

每个 case 创建一个 sudog，全部挂入对应队列，**同一个 G，3 个 sudog**：
```
ch1.sendq → sudog1(G, elem=&v, isSelect=true)
ch2.recvq → sudog2(G, elem=&v, isSelect=true)
ch3.recvq → sudog3(G, elem=&v, isSelect=true)

G.waiting → sudog1 → sudog2 → sudog3  ← G 自己也维护这条链
```

isSelect=true 是关键标记，后面 CAS 会用到。isSelect=true 标记这个 sudog 是 select 的参与者，目的是告诉 唤醒方 在操作前必须先做 CAS 竞争，而不是直接唤醒 G。

所有 channel 解锁，G 调用 gopark 挂起，M 去执行其他 G。

某个时刻 ch2 来了数据，sender 尝试唤醒 G：通过 CAS 竞争决出唯一赢家。即使 ch1、ch2、ch3 同时就绪，CAS 保证**只有一个 case 能唤醒 G**。

G 被唤醒后，执行与 pause 时**镜像对称**的清理动作：
```
pause 时：lock ch1,ch2,ch3 → 挂入队列 → unlock → gopark
                     ↕ 镜像
resume 时：goready → lock ch1,ch2,ch3 → 从队列移除未赢的 sudog → unlock
```

具体清理：
```
ch2 赢了：
  sudog2 被取出，完成数据拷贝
  重新 lock(ch1), lock(ch3)
  从 ch1.sendq 移除 sudog1
  从 ch3.recvq 移除 sudog3
  unlock(ch1), unlock(ch3)
  释放 sudog1, sudog3 回对象池
```

输掉的 sudog 必须清理，否则 G 已经继续执行了，那些 sudog 还挂在队列里，后续操作会错误地再次唤醒一个已经不在等待的 G。


## 参考
> https://github.com/gophercon/2017-talks/blob/master/KavyaJoshi-UnderstandingChannels/Kavya%20Joshi%20-%20Understanding%20Channels.pdf  
https://www.classcentral.com/course/youtube-gophercon-2017-kavya-joshi-understanding-channels-235585


