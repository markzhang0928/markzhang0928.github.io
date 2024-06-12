---
title: Golang 通道channel源码走读
summary: 本文是对Golang channel的关键源码进行走读，并记录使用时容易犯的一些错误。示例可能来源于网络、各类书籍，对应的Golang版本是1.21.9，仅供个人学习使用。
date: 2024-06-03

authors:
  - admin

tags:
  - Golang channel
  - select、 for-range
  - Data Types
---

## 使用语法和基本示例
1. 超时控制
```go
func waitForStopOrTimeout(stopCh <-chan struct{}, timeout time.Duration) <- chan struct{} {
	stopChWithTimeout := make(chan struct{})
	go func() {
	    select {
		    case <-stopCh: // 收到停止信号
			case <-time.After(timeout): // 等待超时
        }
		close(stopChWithTimeout)
    }()
	return stopChWithTimeout
}
```
该函数返回一个管道，可用于在函数之间传递，但该管道会在指定时间后自动关闭。

2. 控制并发数
```golang
var token = make(chan int, 10) // 创建缓冲型channel, 容量为10

func main() {

	for _, w := range work {
	    go func() {
			token <- 1
			w() // 执行任务，访问第三方动作在w()中
			<-token
        }()   	
    }
}
```
1. 如果把token <- 1放到func外层，就是控制系统goroutine的数量。
2. channel控制子协程的方式，不如waitGroup、Context等更优雅。

## 实现原理

## 数据结构

```golang
// channel 数据结构
type hchan struct {
    qcount   uint           // 当前 channel 中存在多少个元素；
    dataqsiz uint           // 当前 channel 能存放的元素容量；
    buf      unsafe.Pointer // channel 中用于存放元素的环形缓冲区；
    elemsize uint16         // channel 元素类型的大小；
    closed   uint32         // 标识 channel 是否关闭；
    elemtype *_type         // channel 元素类型；
    sendx    uint           // 发送元素进入环形缓冲区的 index；
    recvx    uint           // 接收元素所处的环形缓冲区的 index；
    recvq    waitq          // 因接收而陷入阻塞的协程队列；
    sendq    waitq          // 因发送而陷入阻塞的协程队列；
    
    lock mutex // 用来确保每个读/写操作对 channel 的修改是原子的；
}

// waitq 是 sudog 的双向链表，用于存放因接收而陷入阻塞的协程；
type waitq struct {
    first *sudog       // 队列头部
    last  *sudog       // 队列尾部
}

// sudog是对goroutine的封装
type sudog struct {

	g *g

	next *sudog     // 队列中的下一个节点；
	prev *sudog     // 队列中的上一个节点；
	elem unsafe.Pointer // 读取/写入 channel 的数据的容器;

	acquiretime int64
	releasetime int64
	ticket      uint32

	// 标识当前协程是否处在 select 多路复用的流程中
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool

	// success 表示跟 c通道通信的状态; 
	// true 代表 goroutine已激活，通道 c 中存在数据;
	// false 代表 goroutine 已被唤醒，但通道 c 中不存在数据。
	success bool

	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // 标识与当前 sudog 交互的 chan.
}
```
## 初始化
初始化的过程，主要是在堆上分配内存，并初始化channel的成员变量。
```golang

const (
    maxAlign  = 8
    hchanSize = unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
    debugChan = false
)

func makechan(t *chantype, size int) *hchan {
	elem := t.Elem

	// 编译器代码越界检查
	if elem.Size_ >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.Align_ > maxAlign {
		throw("makechan: bad alignment")
	}
    // 计算 channel 需要的内存大小，以及是否超出最大值
	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// 根据类型，初始 channel，分为 无缓冲型、有缓冲元素为 struct 型、有缓冲元素为 pointer 型 channel;
	var c *hchan
	switch {
	case mem == 0:
		// Channel 容量或元素大小为0
		// 1. 非缓冲型： buf没用，直接指向chan起始地址
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.PtrBytes == 0:
		// 缓冲的 struct 型： 一次性分配好 96 + mem 大小的空间，并且调整 chan 的 buf 指向 mem 的起始位置；
		// （只进行一次内存分配操作）
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 有缓冲的 pointer 型，则分别申请 chan 和 buf 的空间，两者无需连续（两次内存分配操作）
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size) // 循环数组长度
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.Size_, "; dataqsiz=", size, "\n")
	}
	return c // 返回hchan指针
}
```

## 写流程 (发送过程)
### 不加锁快速检测失败
```golang
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {
        if !block {
            return false
        }
		// 对于未初始化的 chan，写入操作会引发死锁；
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }


	if !block && c.closed == 0 && full(c) {
		// 比较巧妙，满足未关闭 和满的条件时，此处可以少获取一次锁，
        return false
    }
	
    lock(&c.lock)
	
    if c.closed != 0 {
        unlock(&c.lock)
		// 对于已关闭的 chan，写入操作会引发 panic.
        panic(plainError("send on closed channel"))
    }

    // ...
}
```
1) 检测 channel 是否为空，如果为空且 block 为 false，则直接返回；
2) 检测 channel 是否为空，如果为空且 block 为 true，则进入 park 状态；（当前goroutine被挂起，永不唤醒）
3) 对于非空 channel，检测 channel 是否为满，如果为满且 block 为 false，则直接返回；
4) 对于已关闭的channel，写入操作会引发 panic.

### 写时存在阻塞读协程

```golang
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	    ...
	    lock(&c.lock)
	    ...
        if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
            send(c, sg, ep, func() { unlock(&c.lock) }, 3)
            return true
        }
}

1) 从阻塞的协程队列中取出一个 goroutine 的封装对象 sudog；
2) 在 send 方法中，会基于 memmove 方法，直接将元素拷贝交给 sudog 对应的 goroutine；
3) 在 send 方法中会完成解锁动作.

func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
    // src在当前的goroutine栈上，dst是另一个goroutine的栈

    // 如果目标地址的栈发生了栈收缩，当读出了sg.elem后，就不能修改真正的dst位置的值。
	// 所以需要再读和写之前加一个内存屏障
    dst := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.Size_)
    // No need for cgo write barrier checks because dst is always
    // Go memory.
    memmove(dst, src, t.Size_)
}

// send函数将元素直接从发送者的栈复制到接受者的栈中，由sendDirect完成复制。
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // 省略一些竞态检查
	...
	// sg.elem指向接收到的值存放的位置，
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g // sudog上绑定的goroutine
	unlockf() // 解锁
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1) // 唤醒接收的goroutine.skip 和打印栈相关
}
```

### 写时无阻塞读协程但环形缓冲区仍有空间
```golang
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // ...
    lock(&c.lock)
    // ...
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx)
        typedmemmove(c.elemtype, qp, ep) // 将ep指向的带发送的元素复制到循环数组中qp指向的位置
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock) // 解锁，返回.
        return true
    }

    // ...
}

// 返回循环队列里第i个元素的地址处
func chanbuf(c *hchan, i uint) unsafe.Pointer {
	return add(c.buf, uintptr(i)*uintptr(c.elemsize))
}

```
1. 对于缓冲型的channel，如果还有缓冲空间。
2. qp指向buf的sendx位置，将当前元素添加到环形缓冲区 sendx 对应的位置；

### 写时无阻塞读协程且环形缓冲区无空间
```golang
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // ...
    lock(&c.lock)

    // ...
    gp := getg() // 获取当前goroutine的指针
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }

	// 构造封装当前 goroutine 的 sudog 对象；
    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    c.sendq.enqueue(mysg) // 把 sudog 添加到当前 channel 的阻塞写协程队列中；
 
    gp.parkingOnChan.Store(true)
	// 当前goroutine被挂起
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

    KeepAlive(ep)

    // someone woke us up.
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    gp.activeStackChans = false
    closed := !mysg.success
    gp.param = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    mysg.c = nil // 去掉 mysg 上绑定的channel
    releaseSudog(mysg)
    if closed {
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
		// 被唤醒后，channel关闭了，导致panic
        panic(plainError("send on closed channel"))
    }
    return true
}
```

## 读流程 (接收过程)

### 读空channel
chanrecv 函数接收channel c的元素并将其写入ep指向的内存地址。

```golang
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // ...
    // 如果是一个空channel
	if c == nil {
		if !block { // 空channel，非阻塞接收，直接返回(false, false)
			return
		}
		// 空channel，阻塞接收，gopark函数挂起当前goroutine，没有机会被唤醒
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
		throw("unreachable")
	}
	// 对于非阻塞接收，且channel为空, 不用获取锁，快速检测失败
	if !block && empty(c) { 
		// 空channel，非阻塞接收，直接返回(false, false)
	    if atomic.Load(&c.closed) == 0 {
		    return
	    }
		// channel 已经被关闭，
		if emptyp(ep) {
		    // 如果要接收数据，返回零值
            if ep != nil {
                typedmemclr(c.elemtype, ep) // 根据类型清理相应地址的内存
            }
            return true, false
		}
	}
```
1) 关闭nil channel，会产生panic
2) empty(c): 用来判断channel是否应该被阻塞，当channel为空时，会被block。

### channel 已关闭且内部无元素
```golang
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    lock(&c.lock)

    if c.closed != 0 {
        if c.qcount == 0 {
            // 关闭的channel，并且循环数组buf里没有元素
			unlock(&c.lock)
            if ep != nil {
				// 从一个已关闭的channel中接收数据，且未忽略返回值，则接收值是该类型的零值。
                typedmemclr(c.elemtype, ep) // 根据类型清理相应地址的内存
		    }
        return true, false
        }
        // The channel has been closed, but the channel's buffer have data.
	} else {
    // 等待发送队列里有 goroutine存在，说明buf是满的
        if sg := c.sendq.dequeue(); sg != nil {
			// 两种情况:
			// 1. 非缓冲型channel
			// 2. 缓冲型channel，且buf队列满了
            recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
            return true, true
        }
    }
}

func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if c.dataqsiz == 0 {
		// 非缓冲型channel，直接从发送者拷贝数据到接收者
        if ep != nil {
            // 直接复制数据，从发送者 的goroutine栈到接收者的goroutine栈
            recvDirect(c.elemtype, sg, ep)
        }
    } else {
		// 缓冲型channel，且buf队列满了
        qp := chanbuf(c, c.recvx)
        // 将接收游标处的数据复制给接收者
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        // 将发送者的数据复制到buf
        typedmemmove(c.elemtype, qp, sg.elem)
        c.recvx++
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
    }
    sg.elem = nil
    gp := sg.g // 取出sudog里的goroutine
    unlockf()
    gp.param = unsafe.Pointer(sg)
    sg.success = true
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    goready(gp, skip+1) // 唤醒接收goroutine(runnable)，等调度器调度
}

```

### 读时有阻塞的写协程

```golang
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    lock(&c.lock)
    if c.closed != 0 {
        ...
    } else {
       if sg := c.sendq.dequeue(); sg != nil {
            // Found a waiting sender. If buffer is size 0, receive value
            // directly from sender. Otherwise, receive from head of queue
            // and add sender's value to the tail of the queue (both map to
            // the same buffer slot because the queue is full).
            recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
            return true, true
        } 
    }
}
```
1) 从阻塞写协程队列中获取到一个写协程；
2) 倘若 channel 无缓冲区，则直接读取写协程元素，并唤醒写协程；
3) 倘若 channel 有缓冲区，则读取缓冲区头部元素，并将写协程元素写入缓冲区尾部后唤醒写写成；
4) 解锁，返回.

### 读时无阻塞写协程且缓冲区有元素
```golang
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // ...
    lock(&c.lock)
    // 缓冲型，buf里有元素，可以正常接收
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp) // 清除掉循环数组里相应位置的值
        c.recvx++  // 接收游标加一
        if c.recvx == c.dataqsiz {
            c.recvx = 0 // 接收游标归零
        }
        c.qcount-- // buf数组里的元素个数减一
        unlock(&c.lock) // 解锁
        return true, true
    }
	if !block { // 非阻塞接收，解锁，返回false(未接收到数据)。
		unlock(&c.lock)
		return false, false
    }
    // ...
}
```
### 读时无阻塞写协程且缓冲区无元素
```golang
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // no sender available: block on this channel.
	// 构造一个sudog
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
    mysg.releasetime = -1
    }
    // 待接收数据的地址保存下来
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg) // 进入channel的等待接收队列

    gp.parkingOnChan.Store(true)
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceBlockChanRecv, 2)

	// 被唤醒后，从这里继续执行扫尾工作
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success 
}
```
1) 构造封装当前 goroutine 的 sudog 对象；
2) 完成指针指向，建立 sudog、goroutine、channel 之间的指向关系；
3) 把 sudog 添加到当前 channel 的阻塞读协程队列中；
4) park 当前协程；
5) 倘若协程从 park 中被唤醒，则回收 sudog（sudog能被唤醒，其对应的元素必然已经被写入）；

## 关闭 channel流程
```golang
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock) // 上锁
	if c.closed != 0 { // 如果 channel 已经关闭
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1 // 修改关闭状态，指示channel已关闭

	var glist gList

	// 将 channel 所有等待接收队列里的 sudog 释放
	for {
		sg := c.recvq.dequeue() // 从接收队列里出队一个元素
		if sg == nil { // 出队完毕，退出循环
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// 将channel等待发送队列里的 sudog 释放
	for {
		// 从发送队列里出队一个sudog
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		// 发送者会panic
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock) // 解锁

	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}

```
1）当关闭一个channel时，会根据sendq 和recvq中阻塞的goroutine 队列进行不同处理。
- 1.1 等待接收者而言，会收到一个相应类型的零值，比如chan int 会收到int类型的0。
- 1.2 等待发送者而言，会直接panic。


### 优雅关闭通道
1. 不要在接收方关闭通道。
2. 不要向已关闭的通道发送数据，不要重复关闭通道。
3. 如果通道有多个发送方，不要关闭通道。

## 总结
1. 读、写、关闭操作的场景:
| 操作      | nil channel | closed channel | not nil, not closed channel                                                   |
|---------|------------|---------------|-------------------------------------------------------------------------------|
| close   | panic      | panic         | 正常关闭                                                                          |
| 读 <- ch | 阻塞(死锁)     | 读到对应类型的零值   | 阻塞或正常读取数据。 <br/>1) 缓冲型channel为空，会阻塞。<br/> 2) 非缓冲型channel没有等待的发送者时，会阻塞。  |
| 写 ch <- | 阻塞(死锁)     | panic         | 阻塞或正常写入数据。<br/> 1) 缓冲型channel buf满时, 会阻塞。 <br/> 2) 非缓冲型channel 没有等待的接收者时，会阻塞。 |

2. 判断channel和 mutex的使用时机，一般遵循如下原则：
- 1. 并行的goroutines 使用互斥锁mutex进行同步控制其共享状态或访问共享资源。(互斥锁确保对资源的独占访问)
- 2. 并发的goroutines 使用通道channel进行协作或所有权转移。(需要channel进行通信，channel是goroutine之间协作的桥梁)

