---
title: Go语言中的sync.Cond
date: 2023-10-30 17:50:46
tags:
  - Go
  - sync
categories:
  - 技术
  - 编程语言
  - Go
---
Go语言并发编程中，经常会用到 sync 包。sync包提供了Once、Mutex、Rwmutex、Cond、Map、Pool、Waitgroup等模块，其中 Cond 模块最不常用，很少见在实际业务中有应用到的。那么今天我们就来介绍一下 Cond 模块以及其应用场景，以帮助大家在处理并发编程相关问题时，能够多一个选择。
# 使用场景
sync.Cond 经常用在多个协程等待事件发生，一个协程通知事件发生的场景。举个例子：  
有一个协程在异步地接收数据，剩下的多个协程必须等待这个协程接收完数据，才能读取到正确的数据。在这种情况下，如果单纯使用通道或互斥锁，那么只能有一个协程可以等待，并读取到数据，没办法通知其他的协程也读取数据。  
sync.Cond 就是用来解决如上这类问题的。
# sync.Cond源码分析
## sync.Cond 定义
sync.Cond 定义如下：
```
type Cond struct {
    noCopy noCopy

    // L is held while observing or changing the condition
    L Locker

    notify  notifyList
    checker copyChecker
}
```
每个 Cond 实例都会关联一个锁 L（互斥锁 *Mutex，或读写锁 *RWMutex），当修改条件或者调用 Wait 方法时，必须加锁。
## 常用方法
### NewCond 创建实例
```
// NewCond returns a new Cond with Locker l.
func NewCond(l Locker) *Cond {
    return &Cond{L: l}
}
```
通过调用 NewCond创建一个 Cond 实例，注意，创建 Cond 实例时需要关联一个锁。
### Wait 等待
```
// Wait atomically unlocks c.L and suspends execution
// of the calling goroutine. After later resuming execution,
// Wait locks c.L before returning. Unlike in other systems,
// Wait cannot return unless awoken by Broadcast or Signal.
//
// Because c.L is not locked while Wait is waiting, the caller
// typically cannot assume that the condition is true when
// Wait returns. Instead, the caller should Wait in a loop:
//
//    c.L.Lock()
//    for !condition() {
//        c.Wait()
//    }
//    ... make use of condition ...
//    c.L.Unlock()
func (c *Cond) Wait() {
    c.checker.check()
    t := runtime_notifyListAdd(&c.notify)
    c.L.Unlock()
    runtime_notifyListWait(&c.notify, t)
    c.L.Lock()
}
```
我们翻译一下 Wait 方法的注释：  
Wait 会原子性地释放锁 c.L，并将当前协程挂起，也就是当前协程会阻塞在 Wait 方法里。当其他协程调用了 Broadcast 或 Signal 时，当前协程被唤醒，然后 Wait 方法会重新给 c.L 加锁，并且继续执行 Wait 后面的代码。  
由于当前协程被唤醒时，condition 并不一定符合要求，因此需要再次判断一下condition，所以推荐使用 `for !condition()` 而非 `if`。
### Signal 唤醒单个协程
```
// Signal wakes one goroutine waiting on c, if there is any.
//
// It is allowed but not required for the caller to hold c.L
// during the call.
//
// Signal() does not affect goroutine scheduling priority; if other goroutines
// are attempting to lock c.L, they may be awoken before a "waiting" goroutine.
func (c *Cond) Signal() {
    c.checker.check()
    runtime_notifyListNotifyOne(&c.notify)
}
```
Signal() 会唤醒一个正在等待的协程（如果有的话），调用者在调用期间可以持有 c.L，但这不是必需的。
### Broadcast 广播唤醒所有协程
```
// Broadcast wakes all goroutines waiting on c.
//
// It is allowed but not required for the caller to hold c.L
// during the call.
func (c *Cond) Broadcast() {
    c.checker.check()
    runtime_notifyListNotifyAll(&c.notify)
}
```
Broadcast() 会唤醒所有正在等待的协程，同样调用者在调用期间可以持有 c.L，但这不是必需的。
# 使用示例
下面是一个简单的使用 sync.Cond 的示例：一个协程写入数据，另外几个协程等待并读取数据。
```
package main

import (
    "log"
    "sync"
    "time"
)

var done = false

func read(name string, c *sync.Cond) {
    c.L.Lock()
    for !done {
        log.Println(name, "waiting")
        c.Wait()
    }
    log.Println(name, "start reading")
    c.L.Unlock()
}

func main() {
    c := sync.NewCond(&sync.Mutex{})
    // 启动3个reader
    go read("reader1", c)
    go read("reader2", c)
    go read("reader3", c)
    time.Sleep(time.Second)
    // 开始写数据
    log.Println("start writing")
    done = true
    log.Println("write done")
    // 唤醒一个协程
    log.Println("wake one")
    c.Signal()
    // 由于已经done，所以reader4并不会进入wait
    go read("reader4", c)
    time.Sleep(time.Second)
    // 唤醒其他所有协程
    log.Println("wake all")
    c.Broadcast()
    time.Sleep(time.Second * 3)
}
```
运行结果如下：
>2023/10/31 17:11:16 reader1 waiting
>2023/10/31 17:11:16 reader2 waiting
>2023/10/31 17:11:16 reader3 waiting
>2023/10/31 17:11:17 start writing
>2023/10/31 17:11:17 write done
>2023/10/31 17:11:17 wake one
>2023/10/31 17:11:17 reader1 start reading
>2023/10/31 17:11:17 reader4 start reading
>2023/10/31 17:11:18 wake all
>2023/10/31 17:11:18 reader3 start reading
>2023/10/31 17:11:18 reader2 start reading