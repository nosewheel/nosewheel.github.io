---
title: Go语言中的defer
date: 2023-10-24 21:34:20
tags:
  - Go
  - defer
categories:
  - 技术
  - 编程语言
  - Go
---
defer的功能是在作用域结束之后执行收尾的函数，defer的行为可以总结为如下四条：  
1. defer函数的参数，是在defer函数被定义的时候就已经明确了。 
2. defer函数是在return执行之后运行的。
3. 函数运行过程中出现panic，defer函数也会被执行。
4. defer函数的执行顺序是后进先出。

下面我们通过一些例子，来详细介绍下这四条规则。  
1）defer函数的参数，是在defer函数被定义的时候就已经明确了。   
```
package main

import "fmt"

func main() {
    a()
}
func a() {
    i := 0
    defer fmt.Println(i)
    i++
    return
}
```
上面示例输出结果：0。defer调用fmt.Println(i)时，会对函数中引用的外部参数进行拷贝，所以i++操作并没有影响到defer里的i值。  
再看下面这个例子：  
```
package main

import "fmt"

func main() {
    a()
}
func a() {
    i := 0
    defer func() {
        fmt.Println(i)
    }()
    i++
    return
}
```
这个示例输出结果是：1。为什么跟上一个示例输出结果不一样呢？因为这里defer后面是一个匿名函数，而i并不是该匿名函数的外部参数，这里大家一定要注意。  
  
2）defer函数是在return执行之后运行的。  
```
package main

import "fmt"

func main() {
    fmt.Println("main", a())
}
func a() int {
    i := 0
    defer func() {
        i++
        fmt.Println("defer", i)
    }()
    return i
}
```
该示例运行结果是：
>defer 1
>main 0

虽然在defer里对i进行了操作，但是因为defer是在return执行之后运行的，所以并不会影响返回值。  
  
3）函数运行过程中出现panic，defer函数也会被执行。
```
package main

import "fmt"

func main() {
    fmt.Println("main", a())
}
func a() int {
    i := 0
    defer func() {
        i++
        fmt.Println("defer1", i)
    }()
    panic("I'm panic")
    fmt.Println("after panic")
    defer func() {
        i++
        fmt.Println("defer2", i)
    }()
    return i
}
```
该示例运行结果是：
>defer1 1
>panic: I'm panic
>
>goroutine 1 [running]:
>main.a()
>        /Users/chenzeping/go/src/gostudy/defer/main.go:14 +0x68
>main.main()
>        /Users/chenzeping/go/src/gostudy/defer/main.go:6 +0x1c
>exit status 2

可以看到，在panic之前的defer被执行了，然后才是panic，panic之后的代码都没有执行。至于为什么defer会被执行，我在后面的博客[Go语言异常处理机制](/2023/10/25/golang-exception-handle)中会详细解答。  
  
4）defer函数的执行顺序是后进先出。  
```
package main

import "fmt"

func main() {
    a()
}
func a() {
    defer fmt.Println("defer1")
    defer fmt.Println("defer2")
    return
}
```
该示例运行结果是：
>defer2
>defer1

可以看到，先执行了defer2，后执行了defer1。这个也好理解，每次defer都会把一个函数压入栈中，函数返回前再把延迟的函数从栈中取出并执行，顺序是后进先出。  
