---
title: Goroutine 上下文context源码走读
summary: 本文是对Goroutine context的关键源码进行走读，并记录使用时容易犯的一些错误。示例可能来源于网络、各类书籍，对应的Golang版本是1.21.9，仅供个人学习使用。
date: 2024-06-12

authors:
  - admin

tags:
  - Golang Context
  - Data Types
---

golang context上下文，主要用来在多个goroutine之间传递上下文信息，实现取消信号、超时/截止时间、k-v信息传递等goroutine生命周期的控制。

## 使用语法和基本示例
1. 请求级别的上下文传递，比如: 使用context传递 服务端 requestId。
```golang
func main() {
  ctx := context.Background()
  process(ctx)

  ctx = context.WithValue(ctx, "traceId", "qcrao-2020")
  process(ctx)
}

func process(ctx context.Context) {
  traceId, ok := ctx.Value("traceId").(string)
  if ok {
    fmt.Printf("process over.trace_id=%s\n", traceId)
  } else {
    fmt.Printf("process over.no_trace_id\n")
  }
}
```

2. 定时取消: withTimeOut函数返回的context和cancelFun分开
```golang
func main() {
  ctx, cancel := context.WithTimeout(context.Background, 1 * time.Second) // 1 or 5 time.Second
  defer cancel()

  ids := getResultFromWeb(ctx)
  fmt.Prinln(ids)
}

func getResultFromWeb(ctx context.Context) (res []int64) {
  select {
  case <- time.After(3* time.Second):
    return []int64{100, 200, 300}
  case <- ctx.Done():
    return []int64{1, 2, 3}
  }
}

```

3. 防止泄漏: 当goroutine发生异常阻塞时，如果没有设置TimeOut或者主动cancel, 就会发生goroutine泄漏。

## 数据结构

```golang
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```
context的实现定义了接口，实现这些接口函数的类均被当作context的一种。

• Deadline()：该方法返回一个deadline代表context 的过期时间，标识是否已设置deadline的bool值；

• Done()：该方法返回一个context 的只读channel，需要在select-case语句中使用，如 case <- context.Done(): 

• Err()：返回context被关闭的原因；关闭原因由context控制，不需要用户设置。
  比如：1. 因deadline关闭: context deadline exceeded; 2. 因主动关闭: context canceled; 

• Value()：该方法用于根据key值返回查询map中的value， context 中的对应 key 的值.

context 源码主要包括几个关键结构体，分别实现了Context类和 canceler类

{{< figure src="context_classes.png" caption="context整体类图" theme="light" >}}

### 1. emptyCtx: context.Backgroud() && context.TODO() 实现了emptyCtx类。

```golang
type emptyCtx struct{}

func (emptyCtx) Deadline() (deadline time.Time, ok bool) {
  return
}

func (emptyCtx) Done() <-chan struct{} {
  return nil
}

func (emptyCtx) Err() error {
  return nil
}

func (emptyCtx) Value(key any) any {
  return nil
}

type backgroundCtx struct{ emptyCtx } // 作用在顶层，作为所有context的根节点

func (backgroundCtx) String() string {
  return "context.Background"
}

type todoCtx struct{ emptyCtx }

func (todoCtx) String() string {
  return "context.TODO"
}
```
• emptyCtx 是一个空的 context，本质上类型为一个整型；

• Deadline() 方法会返回一个公元元年时间以及 false 的 flag，标识当前 context 不存在过期时间；

• Done() 方法返回一个 nil 值，用户无论往 nil 中写入或者读取数据，均会陷入阻塞；

• Err() 方法返回的错误永远为 nil；

• Value() 方法返回的 value 同样永远为 nil.

### 2. cancelCtx: 

```golang
type cancelCtx struct {
  Context

  mu       sync.Mutex            // protects following fields
  done     atomic.Value          // 实际类型为 chan struct{}，即用以反映 cancelCtx 生命周期的通道；
  children map[canceler]struct{} // 记录由Context派生出来的所有child，此Context取消会把其中所有child都取消掉
  err      error                 // set to non-nil by the first cancel call
  cause    error                 // set to non-nil by the first cancel call
}


type canceler interface {
  cancel(removeFromParent bool, err, cause error)
  Done() <-chan struct{}
}

```
• Done() 方法返回一个channel, channel一旦关闭，就会读出零值。cancelCtx.done的值一般经历如下三个阶段: nil -> chan struct{} -> closed chan

• Err() 方法返回一个error告知context被关闭的原因即可，cancelCtx来说只需要返回成员变量err。

• Value() 方法返回context中对应key的value值。如果key指定为&cancelCtxKey, 则返回cancelCtx自身的指针；

• 未实现Deadline()方法，仅嵌套了一个带有deadline方法的Context。


```golang

func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
  // 必须要传err
  if err == nil {
    panic("context: internal error: missing cancel error")
  }
  if cause == nil {
    cause = err // 设置cause=err 说明关闭原因
  }
  c.mu.Lock()
  if c.err != nil {
    c.mu.Unlock()
    return // already canceled
  }
  // 给err字段赋值
  c.err = err
  c.cause = cause
  // 关闭channel，通知其他协程
  d, _ := c.done.Load().(chan struct{})
  if d == nil {
    c.done.Store(closedchan)
  } else {
    close(d)
  }

  for child := range c.children { // 遍历所有children, 逐个调用cancel方法取消子节点
    // NOTE: acquiring the child's lock while holding parent's lock.
    child.cancel(false, err, cause)
  }
  // 将子节点置空
  c.children = nil
  c.mu.Unlock()

  if removeFromParent {
    // 从父节点context中移除自己本身的context
    removeChild(c.Context, c)
  }
}
```
•  cancel方法的作用是关闭c.done；递归地取消所有子节点；从父节点从删除自己；

•  作用：通过关闭channel，将取消信号传递给它的所有子节点; 

•  goroutine收到取消信号的方式，是select语句中读c.done分支被选中；

注意:
1) context本身并没有取消函数，取消函数只能由外层函数调用，防止子节点context调用取消函数，从而严格控制信息的流向：由父节点context流向子节点context。


### 3. timeCtx: 用来表示自动cancel的最终时间，
```golang
type timerCtx struct {
  cancelCtx
  timer *timer.timer // 指定最后期限，自动触发cancel的定时器

  deadline time.Time // 指定最长存活时间
}
```
• Deadline()方法 返回timeCtx.deadline过期时间, 而timerCtx.deadline 是WithDeadline() 或 WithTimeout()方法设置的。

• cancel()基本继承cancelCtx, 只需额外关闭timer，当timerCtx被关闭时，timerCtx.cancelCtx.err存储关闭原因。

### 4. valueCtx: 
只能存一个kv对，不支持key去重，当前 valueCtx 的 key 不等于用户传入的 key，则从 parent context 中依次向上寻找。


## Go官方文档的基本建议
1. 不要将 Context 塞到结构体里。直接将 Context 类型作为函数的第一参数，而且一般都命名为 ctx。
2. 不要向函数传入一个 nil 的 context，如果你实在不知道传什么，标准库给你准备好了一个 context：todo。
3. 不要把本应该作为函数参数的类型塞到 context 中，context 存储的应该是一些共同的数据。例如：登陆的 session、cookie 等。(业务相关的值不建议使用context进行传递)
4. 同一个 context 可能会被传递到多个 goroutine，context 是并发安全的。

