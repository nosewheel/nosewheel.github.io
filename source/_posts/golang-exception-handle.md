---
title: Go语言异常处理机制
date: 2023-10-25 10:34:51
tags:
  - Go
  - panic
  - recover
  - defer
categories:
  - 技术
  - 编程语言
  - Go
---
Go语言舍弃了传统的 try...catch 类似的异常处理机制，但是我们仍然可以通过Go语言提供了 panic 和 recover 函数，配合 defer 语句灵活地处理运行时的异常。  
# 名词介绍
## panic
Go语言的内置方法，能够改变程序的控制流。当函数调用了panic，函数会停止运行，但是defer函数仍然会运行，程序会在当前panic的goroutine全部退栈以后crash。
## recover
Go语言的内置方法，用于恢复发生panic的goroutine的控制。如果当前goroutine将要发生panic的话，recover会捕获这个panic，并恢复正常执行。
## defer
Go语言的关键字，用来延迟执行函数的，延迟的发生是在调用函数的return之后。关于defer，我在博客[Go语言中的defer](/article/golang-defer)中详细介绍过。而在这里为什么会提到defer呢？这是因为recover只在defer函数中生效。
# 常见用法
如下是一个常见的捕获异常的例子
```
package main

import "fmt"

func main() {
    defer func() {
        if err := recover(); err != nil {
            fmt.Println("catch panic:", err)
        }
    }()
    panic("I'm panic")
}
```
输出结果为：
>catch panic: I'm panic

需要注意的几个点：  
1）recover只在defer函数中生效  
下面这几种写法，都不会捕获panic
```
package main

import "fmt"

func main() {
    if err := recover(); err != nil {
        fmt.Println("catch panic:", err)
    }
    panic("I'm panic")
}
```
该示例输出结果：
>panic: I'm panic
>
>goroutine 1 [running]:
>main.main()
>        /Users/chenzeping/go/src/gostudy/panic/main.go:9 +0x3c
>exit status 2

上面示例中recover()在panic之前运行，此时panic还未发生，肯定不会捕获panic。  
再看下面这个例子
```
package main

import "fmt"

func main() {
    panic("I'm panic")
    if err := recover(); err != nil {
        fmt.Println("catch panic:", err)
    }
}
```
该示例输出结果：
>panic: I'm panic
>
>goroutine 1 [running]:
>main.main()
>        /Users/chenzeping/go/src/gostudy/panic/main.go:6 +0x2c
>exit status 2

上面示例中recover()虽然写在了panic之后，但是由于panic后，后面的代码不会被执行，所以程序执行不到recover()，也就无法捕获panic。  

2）recover只能捕获当前goroutine的panic
看下面这个例子
```
package main

import (
    "fmt"
    "time"
)

func main() {
    a()
    time.Sleep(2 * time.Second)
}

func a() {
    defer func() {
        fmt.Println("in defer")
        if err := recover(); err != nil {
            fmt.Println("catch panic:", err)
        }
    }()
    go func() {
        time.Sleep(1 * time.Second)
        panic("I'm panic")
    }()
}
```
输出结果为：
>in defer
>panic: I'm panic
>
>goroutine 18 [running]:
>main.a.func2()
>        /Users/chenzeping/go/src/gostudy/panic/main.go:22 +0x38
>created by main.a
>        /Users/chenzeping/go/src/gostudy/panic/main.go:20 +0x40
>exit status 2

从上面例子可以看出，其他goroutine中的panic并没有被recover捕获，从而最终导致程序崩溃。
