---
title: Golang中的单例模式
date: 2023-10-19 21:12:44
tags:
  - Go
  - 单例
  - 并发
  - Go内存模型
categories:
  - 技术
  - 编程语言
  - Go
---
Go语言中如何实现一个单例？很多Go语言学习者可能觉得这是一个很简单的问题，闭着眼都能写出来。但是其代码实现往往存在不少问题，下面先看一下一些常见的错误。
# 常见错误
## 没考虑并发安全
示例代码：
```
type User struct {
    Name string
}

var instance *User

func GetInstance() *User {
    if instance == nil {
        instance = &User{
            Name: "xiaoming",
        }
    }
    return instance
}
```
上面示例中，多个goroutine有可能同时执行第一个检查，然后instance就会被执行多次赋值操作，在某些instance只能被赋值一次的业务场景，该操作就可能导致与期望不一致。
## 忽略代码非原子操作问题
针对上面示例存在的并发问题，部分开发人员想我加个锁，问题不就解决了吗，于是将上面示例代码修改成如下：
```
type User struct {
    Name string
}

var (
    lock     sync.Mutex
    instance *User
)

func GetInstance() *User {
    if instance == nil {
        lock.Lock()
        defer lock.Unlock()
        if instance == nil {
            instance = &User{
                Name: "xiaoming",
            }
        }
    }
    return instance
}
```
如上示例代码看似已经很完美了，但是当我们用go run -race检查时，会提示这段代码有DATA RACE的警告：
```
==================
WARNING: DATA RACE
Write at 0x00010507e6a0 by goroutine 6:
  main.GetInstance()
      /Users/chenzeping/go/src/gostudy/race/go_race_demo.go:22 +0xf4
  main.main.func1()
      /Users/chenzeping/go/src/gostudy/race/go_race_demo.go:32 +0x20

Previous read at 0x00010507e6a0 by goroutine 8:
  main.GetInstance()
      /Users/chenzeping/go/src/gostudy/race/go_race_demo.go:18 +0x30
  main.main.func3()
      /Users/chenzeping/go/src/gostudy/race/go_race_demo.go:38 +0x20

Goroutine 6 (running) created at:
  main.main()
      /Users/chenzeping/go/src/gostudy/race/go_race_demo.go:31 +0x28

Goroutine 8 (running) created at:
  main.main()
      /Users/chenzeping/go/src/gostudy/race/go_race_demo.go:37 +0x40
==================
```
大家可以亲自试一下，go_race_demo.go完整代码如下：
```
package main

import (
    "fmt"
    "sync"
)

type User struct {
    Name string
}

var (
    lock     sync.Mutex
    instance *User
)

func GetInstance() *User {
    if instance == nil {
        lock.Lock()
        defer lock.Unlock()
        if instance == nil {
            instance = &User{
                Name: "xiaoming",
            }
        }
    }
    return instance
}

func main() {
    go func() {
        fmt.Println("goroutine A", GetInstance())
    }()
    go func() {
        fmt.Println("goroutine B", GetInstance())
    }()
    go func() {
        fmt.Println("goroutine C", GetInstance())
    }()
}
```
之所以会出现DATA RACE警告，是因为CPU在执行instance=&User{Name:"xiaoming"}这行代码时，并不是原子操作，这个赋值可能是会有几步指令，比如
1. 先new一个User
2. 然后设置Name=xiaoming
3. 最后把了new的对象赋值给instance

而且多个指令执行时，有可能会是乱序的，如果发生了乱序，可能会变成
1. 先了new一个User
2. 然后再赋值给instance
3. 最后再设置Name=xiaoming

goroutine A进来时拿到锁，然后执行instance=&User{Name:"xiaoming"}这句代码，这个时候有可能刚执行完指令2，还未执行指令3时，goroutine B对instance是否为nil进行了判断，发现非nil，就直接将instance的数据返回了，而此时的instance是个半初始化状态，这时就会有问题。
# 正确姿势
解决上面的问题，我们可以通过原子化加载并设置一个标志flag，该标志表明我们是否已初始化instance，改造后代码如下：
```
type User struct {
    Name string
}

var (
    lock     sync.Mutex
    instance *User
    flag     uint32
)

func GetInstance() *User {
    if atomic.LoadUint32(&flag) != 1 {
        lock.Lock()
        defer lock.Unlock()
        if instance == nil {
            instance = &User{
                Name: "xiaoming",
            }
            atomic.StoreUint32(&flag, 1)
        }
    }
    return instance
}
```
这里，我们主要是通过atomic.store和lock来保证flag和instance的修改对其他的goroutine可见，通过atomic.LoadUint32(&flag)和double check来保证instance只会被初始化一次。  
但是，这看起来有点繁琐，其实Go语言标准库sync已经为我们提供了实现goroutine同步比较好的方式，通过`sync.Once`来实现，示例代码如下：
```
type User struct {
    Name string
}

var (
    once sync.Once
    instance *User
)

func GetInstance() *User {
    once.Do(func() {
        instance = &User{
            Name: "xiaoming",
        }
    })
    return instance
}
```
因此，我们可以首选`sync.Once`来实现单例模式。
