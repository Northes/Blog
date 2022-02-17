---
title: "Go 内存模型"
date: 2022-01-13T22:10:50+08:00
draft: false
author : "Northes"
description: "眼见不一定为实 - Go内存模型详解"
tags: [翻译","Golang"]
---

*翻译整理自 https://go.dev/ref/mem*


## 介绍
如何保证在一个 `goroutine` 中看到在另一个 `goroutine` 修改的变量的值，这篇文章进行了详细说明。

## 建议
如果程序中修改数据时有其他 `goroutine` 同时读取，那么必须将读取串行化。为了串行化访问，请使用 `channel` 或其他同步原语，例如 `sync` 和 `sync/atomic` 来保护数据。

## 先行发生
在一个 `goroutine` 中，读和写一定是按照程序中的顺序执行的。即编译器和处理器只有在不会改变这个 `goroutine` 的行为时才可能修改读和写的执行顺序。由于重排，不同的 `goroutine` 可能会看到不同的执行顺序。例如，一个 `goroutine` 执行 a = 1;b = 2; 另一个 `goroutine` 可能看到 b 在 a 之前更新。

为了说明读和写的必要条件，我们定义了**先行发生**（Happens Before）-- Go 程序中执行内存操作的偏序。

- 如果事件 `e1` 发生在 `e2` 前，我们可以说 `e2` 发生在 `e1` 后。
- 如果 `e1` 不发生在 `e2` 前也不发生在 `e2` 后，我们就说 `e1` 和 `e2` 是并发的。

***在单独的goroutine中先行发生的顺序即是程序中表达的顺序。***

当下面条件满足时，对变量v的读操作r是被允许看到对v的写操作w的：

1. r不先行发生于w
2. 在w后r前没有对v的其他写操作

为了保证对变量v的读操作r看到对v的写操作w,要确保w是r允许看到的唯一写操作。即当下面条件满足时，r 被保证看到w：

1. w先行发生于r
2. 其他对共享变量v的写操作要么在w前，要么在r后。

这一对条件比前面的条件更严格，需要没有其他写操作与w或r并发发生。

单独的 `goroutine` 中没有并发，所以上面两个定义是相同的：读操作r看到最近一次的写操作w写入v的值。

当多个 `goroutine` 访问共享变量v时，它们必须使用同步事件来建立先行发生这一条件来保证读操作能看到需要的写操作。 对变量v的零值初始化在内存模型中表现的与写操作相同。 对大于一个字的变量的读写操作表现的像以不确定顺序对多个一字大小的变量的操作。

## 同步
### 初始化
程序的初始化在单独的 `goroutine` 中进行，但这个 `goroutine` 可能会创建出并发执行的其他 `goroutine`。

如果包p引入（import）包q，那么q的 `init` 函数的结束先行发生于p的所有 `init` 函数开始，`main.main` 函数的开始发生在所有 `init` 函数结束之后

### 创建goroutine
go关键字开启新的 `goroutine`，先行发生于这个 `goroutine` 开始执行，例如下面程序：
```go
var a string

func f() {
    print(a)
}

func hello() {
    a = "hello, world"
    go f()
}
```
调用 `hello` 会在之后的某时刻打印出 "hello, world"（可能在hello返回之后）

### 销毁goroutine
`goroutine` 的退出并不会保证先行发生于程序的任何事件。例如下面程序：
```go
var a string

func hello() {
    go func() { a = "hello" }()
    print(a)
}
```

没有用任何同步操作限制对a的赋值，所以并不能保证其他 `goroutine` 能看到a的变化。实际上，一个激进的编译器可能会删掉整个go语句。 如果想要在一个 `goroutine` 中看到另一个 `goroutine` 的执行效果，请使用`锁`或者`channel`这种同步机制来建立程序执行的相对顺序。

### channel通信
`channel` 通信是 `goroutine` 同步的主要方法。每一个在特定 `channel` 的发送操作都会匹配到通常在另一个 `goroutine` 执行的接收操作。

在 `channel` 的发送操作先行发生于对应的接收操作完成，例如：
```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```
这个程序能保证打印出 "hello, world"。对a的写先行发生于在c上的发送，先行发生于在c上的对应的接收完成，先行发生于 `print`。

***对channel的关闭先行发生于接收到零值，因为channel已经被关闭了***

在上面的例子中，将c <- 0替换为close(c)也会产生同样的结果。

***无缓冲channel的接收先行发生于发送完成***

如下程序（和上面类似，只交换了对 `channel` 的读写位置并使用了非缓冲 `channel` ）：

```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}

func main() {
	go f()
	c <- 0
	print(a)
}
```

此程序也能保证打印出 "hello, world"。对a的写先行发生于从c接收，先行发生于向c发送完成，先行发生于 `print`。

如果是带缓冲的 `channel`（例如c = make(chan int, 1)），程序不保证打印出 "hello, world"(可能打印空字符，程序崩溃或其他行为)。

***在容量为C的channel上的第k个接收先行发生于从这个channel上的第k+C次发送完成。***

这条规则将前面的规则推广到了带缓冲的 `channel` 上。可以通过带缓冲的 `channel` 来实现计数信号量：`channel` 中的元素数量对应着活动的数量，`channel` 的容量表示同时活动的最大数量，发送元素获取信号量，接收元素释放信号量，这是限制并发的通常用法。

下面程序为 `work` 中的每一项开启一个 `goroutine`，但这些 `goroutine` 通过有限制的 `channel` 来确保最多同时执行三个工作函数（w）。

```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

### 锁
`sync` 包实现了两个锁的数据类型 `sync.Mutex` 和 `sync.RWMutex`。

***对任意的sync.Mutex或sync.RWMutex变量l和n < m，n次调用l.Unlock()先行发生于m次l.Lock()返回***

下面程序：
```go
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```
能保证打印出 "hello, world"。第一次调用 `l.Unlock()`（在f()中）先行发生于main中的第二次 `l.Lock()` 返回, 先行发生于 `print`。

***对于sync.RWMutex变量l，任意的函数调用l.RLock满足第n次l.RLock后发生于第n次调用l.Unlock，对应的l.RUnlock先行发生于第n+1次调用l.Lock。***

### Once
`sync` 包的 `Once` 为多个 `goroutine` 提供了安全的初始化机制。能在多个线程中执行 `once.Do(f)`，但只有一个f()会执行，其他调用会一直阻塞直到f()返回。
通过执行先行发生（指f()返回）于其他的返回。
如下程序：
```go
var a string
var once sync.Once

func setup() {
	a = "hello, world"
}

func doprint() {
	once.Do(setup)
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```


调用 `twoprint` 会打印"hello, world"两次。`setup` 只在第一次 `doprint` 时执行。

## 错误的同步方法
注意，读操作r可能会看到并发的写操作w。即使这样也不能表明r之后的读能看到w之前的写。
如下程序：
```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```
g可能先打印出2然后是0。
这个事实证明一些旧的习惯是错误的。
双重检查锁定是为了避免同步的资源消耗。例如 `twoprint` 程序可能会错误的写成：
```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

在 `doprint` 中看到 `done` 被赋值并不能保证看到对a赋值。此程序可能会错误地输出空字符而不是 "hello, world" 。
另一个错误的习惯是忙等待，例如：

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

和之前程序类似，在 `main` 中看到 `done` 被赋值不能保证看到a被赋值，所以此程序也可能打印出空字符。更糟糕的是因为两个线程间没有同步事件，在 `main` 中可能永远不会看到 `done` 被赋值，所以`main` 中的循环不保证能结束。
对程序做一个微小的改变：

```go
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

即使main看到了 `g != nil` 并且退出了循环，也不能保证看到 `g.msg` 的初始化值。
在上面所有的例子中，解决办法都是相同的：

**明确地使用同步**

*参考 https://www.jianshu.com/p/5e44168f47a3*