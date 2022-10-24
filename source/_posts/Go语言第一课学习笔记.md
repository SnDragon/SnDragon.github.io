---
title: Go语言第一课学习笔记
date: 2022-10-23 23:41:04
tags: ["Go"]
---

## Go历史
Go语言之父: Rob Pike

很多 Go 语言初学者经常称这门语言为 Golang，其实这是不对的：“Golang”仅应用于命名 Go 语言官方网站，而且当时没有用 go.com 纯粹是这个域名被占用了而已。

![image](https://static001.geekbang.org/resource/image/04/fa/042843f49a53faa6e208c76ef6ed75fa.png?wh=1920x505)


Go 1.4 引入 internal 包机制，增加了 internal 目录

## 设计哲学
Go 语言的设计哲学总结为五点：简单、显式、组合、并发和面向工程

如果说一门编程语言是“自带电池”，则说明这门语言标准库功能丰富，多数功能不需要依赖外部的第三方包或库，Go 语言恰恰就是这类编程语言。

<!--more-->
## 典型目录结构
Go 可执行程序项目的典型结构布局
```go

$tree -F exe-layout 
exe-layout
├── cmd/
│   ├── app1/
│   │   └── main.go
│   └── app2/
│       └── main.go
├── go.mod
├── go.sum
├── internal/
│   ├── pkga/
│   │   └── pkg_a.go
│   └── pkgb/
│       └── pkg_b.go
├── pkg1/
│   └── pkg1.go
├── pkg2/
│   └── pkg2.go
└── vendor/
```
Go 库项目的典型结构布局:
```go
$tree -F lib-layout 
lib-layout
├── go.mod
├── internal/
│   ├── pkga/
│   │   └── pkg_a.go
│   └── pkgb/
│       └── pkg_b.go
├── pkg1/
│   └── pkg1.go
└── pkg2/
    └── pkg2.go
```


## 构建演进
Go 程序的构建过程就是确定包版本、编译包以及将编译后得到的目标文件链接在一起的过程
Go 语言的构建模式历经了三个迭代和演化过程:
* GOPATH
* 1.5 版本的 Vendor 机制
* Go Module

![image](https://static001.geekbang.org/resource/image/49/1b/49eb7aa0458d8ec6131d9e5661155f1b.jpeg?wh=1920x1080)

A 明明说只要求 C v1.1.0，B 明明说只要求 C v1.3.0。所以 Go 会在该项目依赖项的所有版本中，选出符合项目整体要求的“最小版本”。这个例子中，C v1.3.0 是符合项目整体要求的版本集合中的版本最小的那个，于是 Go 命令选择了 C v1.3.0，而不是最新最大的 C v1.7.0。并且，Go 团队认为“最小版本选择”为 Go 程序实现持久的和可重现的构建提供了最佳的方案。

![image](https://static001.geekbang.org/resource/image/45/d3/45bdecc5fa873e06893d6658e447a8d3.jpeg?wh=1920x1080)


Go 命令使用==最小版本选择机制==进行包依赖版本选择


查询依赖包版本:
```go
$go list -m -versions github.com/sirupsen/logrus
github.com/sirupsen/logrus v0.1.0 v0.1.1 v0.2.0 v0.3.0 v0.4.0 v0.4.1 v0.5.0 v0.5.1 v0.6.0 v0.6.1 v0.6.2 v0.6.3 v0.6.4 v0.6.5 v0.6.6 v0.7.0 v0.7.1 v0.7.2 v0.7.3 v0.8.0 v0.8.1 v0.8.2 v0.8.3 v0.8.4 v0.8.5 v0.8.6 v0.8.7 v0.9.0 v0.10.0 v0.11.0 v0.11.1 v0.11.2 v0.11.3 v0.11.4 v0.11.5 v1.0.0 v1.0.1 v1.0.3 v1.0.4 v1.0.5 v1.0.6 v1.1.0 v1.1.1 v1.2.0 v1.3.0 v1.4.0 v1.4.1 v1.4.2 v1.5.0 v1.6.0 v1.7.0 v1.7.1 v1.8.0 v1.8.1
```

降级依赖包版本:
1. 
```go
go get github.com/sirupsen/logrus@v1.7.0
```
2. 
```go
go mod edit -require=github.com/sirupsen/logrus@v1.7.0 
```
go mod vendor 命令在 vendor 目录下，创建了一份这个项目的依赖包的副本，并且通过 vendor/modules.txt 记录了 vendor 下的 module 以及版本

如果我们要基于 vendor 构建，而不是基于本地缓存的 Go Module 构建，我们需要在 go build 后面加上 -mod=vendor 参数。

在 Go 1.14 及以后版本中，如果 Go 项目的顶层目录下存在 vendor 目录，那么 go build 默认也会优先基于 vendor 构建，除非你给 go build 传入 -mod=mod 的参数。

列出所有依赖:
```go
go list -m all
```


## 初始化顺序
Go 在包中按照“常量 -> 变量 -> init 函数”的顺序先对 pkg3 包进行初始化


## 监听操作系统信号
```go
osChan := make(chan os.Signal, 1)
signal.Notify(osChan, syscall.SIGINT, syscall.SIGTERM)
<-osChan
```

## 变量
包级变量只能使用带有 var 关键字的变量声明形式，不能使用短变量声明形式

变量声明:
![image](https://static001.geekbang.org/resource/image/9c/77/9c663f2d88e2e53a8184669bf2338077.jpg?wh=1980x1080)

利用工具检测变量遮蔽问题
```go
go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest
go vet -vettool=$(which shadow) -strict xxx.go
```

```go
type rune = int32
```

Go 字符串类型的内部表示
```go

// $GOROOT/src/reflect/value.go

// StringHeader是一个string的运行时表示
type StringHeader struct {
    Data uintptr
    Len  int
}
```

## 迭代形式
Go 有两种迭代形式：常规 for 迭代与 for range 迭代
通过这两种形式的迭代对字符串进行操作得到的结果是不同的。
经过常规 for 迭代后，我们获取到的是字符串里字符的 UTF-8 编码中的一个字节
通过 for range 迭代，我们每轮迭代得到的是字符串中 Unicode 字符的码点值，以及该字符在字符串中的偏移值


## 常量
Go 语言在常量方面的创新包括下面这几点：
* 支持无类型常量
* 支持隐式自动转型
* 可用于实现枚举。


> 隐式转型说的就是，对于无类型常量参与的表达式求值，Go 编译器会根据上下文中的类型信息，把无类型常量自动转换为相应的类型后，再参与求值计算，这一转型动作是隐式进行的


位于同一行的 iota 即便出现多次，多个 iota 的值也是一样

如果我们要略过 iota = 0，从 iota = 1 开始正式定义枚举常量，我们可以效仿下面标准库中的代码：
```go

// $GOROOT/src/syscall/net_js.go
const (
    _ = iota
    IPV6_V6ONLY  // 1
    SOMAXCONN    // 2
    SO_ERROR     // 3
)
```

## 类型
函数类型、map 类型自身，以及切片类型是不能作为 map 的 key 类型的。

map 可以自动扩容，map 中数据元素的 value 位置可能在这一过程中发生变化，所以 Go 不允许获取 map 中 value 的地址，这个约束是在编译期间就生效的。


Go 语言不支持这种在结构体类型定义中，递归地放入其自身类型字段的定义方式

但可以拥有自身类型的指针类型、以自身类型为元素类型的切片类型，以及以自身类型作为 value 类型的 map 类型的字段

对于各种基本数据类型来说，它的变量的内存地址值必须是其类型本身大小的整数倍

对于结构体而言，它的变量的内存地址，只要是它最长字段长度与系统对齐系数两者之间较小的那个的整数倍就可以了。但对于结构体类型来说，我们还要让它每个字段的内存地址都严格满足内存对齐要求。

内存对齐:出于对处理器存取数据效率的考虑


在日常定义结构体时，一定要注意结构体中字段顺序，尽量合理排序，降低结构体对内存空间的占

```go

type T struct {
    b byte
    i int64
    u uint16
}

type S struct {
    b byte
    u uint16
    i int64
}

func main() {
    var t T
    println(unsafe.Sizeof(t)) // 24
    var s S
    println(unsafe.Sizeof(s)) // 16
}
```

Go 语言不支持在结构体类型定义中，递归地放入其自身类型字段，但却可以拥有自身类型的指针类型、以自身类型为元素类型的切片类型，以及以自身类型作为 value 类型的 map 类型的字段:


一个类型，它所占用的大小是固定的，因此一个结构体定义好的时候，其大小是固定的。
但是，如果结构体里面套结构体，那么在计算该结构体占用大小的时候，就会成死循环。
但如果是指针、切片、map等类型，其本质都是一个int大小(指针，4字节或者8字节，与操作系统有关)，因此该结构体的大小是固定的，记得老师前几节课讲类型的时候说过，类型就能决定内存占用的大小。
因此，结构体是可以接口自身类型的指针类型、以自身类型为元素类型的切片类型，以及以自身类型作为 value 类型的 map 类型的字段，而自己本身不行。


## label
带 label 的 continue 语句，通常出现于嵌套循环语句中，被用于跳转到外层循环并继续执行外层循环语句的下一个迭代，比如下面这段代码：

```go

func main() {
    var sl = [][]int{
        {1, 34, 26, 35, 78},
        {3, 45, 13, 24, 99},
        {101, 13, 38, 7, 127},
        {54, 27, 40, 83, 81},
    }

outerloop:
    for i := 0; i < len(sl); i++ {
        for j := 0; j < len(sl[i]); j++ {
            if sl[i][j] == 13 {
                fmt.Printf("found 13 at [%d, %d]\n", i, j)
                continue outerloop
            }
        }
    }
}


var gold = 38

func main() {
    var sl = [][]int{
        {1, 34, 26, 35, 78},
        {3, 45, 13, 24, 99},
        {101, 13, 38, 7, 127},
        {54, 27, 40, 83, 81},
    }

outerloop:
    for i := 0; i < len(sl); i++ {
        for j := 0; j < len(sl[i]); j++ {
            if sl[i][j] == gold {
                fmt.Printf("found gold at [%d, %d]\n", i, j)
                break outerloop
            }
        }
    }
}
```


for 语句的常见“坑”与避坑方法:
* 循环变量的重用
* 参与循环的是 range 表达式的副本
* 遍历 map 中元素的随机


无论 default 分支出现在什么位置，它都只会在所有 case 都没有匹配上的情况下才会被执行的

```go
func case1() int {
    println("eval case1 expr")
    return 1
}

func case2() int {
    println("eval case2 expr")
    return 2
}

func switchexpr() int {
    println("eval switch expr")
    return 1
}

func main() {
    switch switchexpr() {
    case case1():
        println("exec case1")
        fallthrough
    case case2():
        println("exec case2")
        fallthrough
    default:
        println("exec default")
    }
}
```
```go
eval switch expr
eval case1 expr
exec case1
exec case2
exec default
```
由于 fallthrough 的存在，Go 不会对 case2 的表达式做求值操作，而会直接执行 case2 对应的代码分支

不带 label 的 break 语句中断执行并跳出的，是同一函数内 break 语句所在的最内层的 for、switch 或 select

type switch里是不能fallthrough的

在 Go 中，变长参数实际上是通过切片来实现的


Go 语言使用一个二元组结构 (Data, Len) 来表示一个 string 类型变量，将 string 类型变量作为函数参数，其传递的开销也是恒定的，不会随着字符串大小的变化而变化。


在 Go 中，panic 主要有两类来源，一类是来自 Go 运行时，另一类则是 Go 开发人员通过 panic 函数主动触发的。无论是哪种，一旦 panic 被触发，后续 Go 程序的执行过程都是一样的，这个过程被 Go 语言称为 ==panicking==。



## defer
defer 使用的几个注意事项:
* 明确哪些函数可以作为 deferred 函数
append、cap、len、make、new、imag 等内置函数都是不能直接作为 deferred 函数的，而 close、copy、delete、print、recover 等内置函数则可以直接被 defer 设置为 deferred 函数。
* 注意 defer 关键字后面表达式的求值时机

defer 关键字后面的表达式，是在将 deferred 函数注册到 deferred 函数栈的时候进行求值的。

* 知晓 defer 带来的性能损耗

在 Go 1.13 前的版本中，defer 带来的开销还是很大的,使用 defer 的函数的执行时间是没有使用 defer 函数的 8 倍左右。
但从 Go 1.13 版本开始，Go 核心团队对 defer 性能进行了多次优化，到现在的 Go 1.17 版本，defer 的开销已经足够小了。

## receiver
receiver 参数的基类型本身不能为指针类型或接口类型

```go


package main

import (
    "fmt"
    "time"
)

type field struct {
    name string
}

func (p *field) print() {
    fmt.Println(p.name)
}

func main() {
    data1 := []*field{{"one"}, {"two"}, {"three"}}
    for _, v := range data1 {
        go v.print()
    }

    data2 := []field{{"four"}, {"five"}, {"six"}}
    for _, v := range data2 {
        go v.print()
    }

    time.Sleep(3 * time.Second)
}
```

```
one
two
three
six
six
six
```

```go

type field struct {
    name string
}

func (p *field) print() {
    fmt.Println(p.name)
}

func main() {
    data1 := []*field{{"one"}, {"two"}, {"three"}}
    for _, v := range data1 {
        go (*field).print(v)
    }

    data2 := []field{{"four"}, {"five"}, {"six"}}
    for _, v := range data2 {
        go (*field).print(&v)
    }

    time.Sleep(3 * time.Second)
}
```

Go 语言规定，*T 类型的方法集合包含所有以 *T 为 receiver 参数类型的方法，以及所有以 T 为 receiver 参数类型的方法。


选择 receiver 参数类型的三个原则:
* 如果 Go 方法要把对 receiver 参数代表的类型实例的修改，反映到原类型实例上，那么我们应该选择T 作为 receiver 参数的类型。
* 如果 receiver 参数类型的 size 较大，以值拷贝形式传入就会导致较大的性能开销，这时我们选择 *T 作为 receiver 类型可能更好些。
* 如果 T 类型需要实现某个接口，那我们就要使用 T 作为 receiver 参数的类型，来满足接口类型方法集合中的所有方法。

```go

type T1 struct{}

func (T1) T1M1()   { println("T1's M1") }
func (*T1) PT1M2() { println("PT1's M2") }

type T2 struct{}

func (T2) T2M1()   { println("T2's M1") }
func (*T2) PT2M2() { println("PT2's M2") }

type T struct {
    T1
    *T2
}

func main() {
    t := T{
        T1: T1{},
        T2: &T2{},
    }

    dumpMethodSet(t)
    dumpMethodSet(&t)
}

main.T's method set:
- PT2M2
- T1M1
- T2M1

*main.T's method set:
- PT1M2
- PT2M2
- T1M1
- T2M1
```
通过输出结果，我们看到了 T 和 *T 类型的方法集合果然有差别的：
* 类型 T 的方法集合 = T1 的方法集合 + *T2 的方法集合类型 
* *T 的方法集合 = *T1 的方法集合 + *T2 的方法集合


* 结构体类型的方法集合包含嵌入的接口类型的方法集合；
* 当结构体类型 T 包含嵌入字段 E 时，*T 的方法集合不仅包含类型 E 的方法集合，还要包含类型 *E 的方法集合。


## Go1.17新特性
* 支持将切片转换为数组指针,转换后的数组==长度==不能大于原切片的长度
```go
b := []int{11, 12, 13}
p := (*[3]int)(b) // 将切片转换为数组类型指针
p[1] = p[1] + 10
fmt.Printf("%v\n", b) // [11 22 13]
```
* Go Module 构建模式的变化:修剪的 module 依赖图(pruned module graph)
![image](https://static001.geekbang.org/resource/image/2d/0b/2da4da70a5yy998bf635209642b5c80b.jpg?wh=1980x1080)
go get 已经不再被用来安装某个命令的可执行文件了 -> go install xxx@version
* Go 编译器的变化
    * 基于寄存器的调用惯例:在 AMD64 架构下率先实现了从基于堆栈的调用惯例到基于寄存器的调用惯例的切换。
    * //go:build 形式的构建约束指示符
    ![image](https://static001.geekbang.org/resource/image/22/21/22c37012e48157bdc9a71110bc314421.png?wh=1920x532)
```
//go:build linux && (386 || amd64 || arm || arm64 || mips64 || mips64le || ppc64 || ppc64le)
//go:build linux && (mips64 || mips64le)
//go:build linux && (ppc64 || ppc64le)
//go:build linux && !386 && !arm
```

## 并发模型
传统语言的并发模型是基于对内存的共享的, 一个符合 CSP 模型的并发程序应该是一组通过输入输出原语连接起来的 P 的集合

Goroutine 调度器的任务也就明确了：将 Goroutine 按照一定算法放到不同的操作系统线程中去执行。

如果 G 被阻塞在某个 channel 操作或网络 I/O 操作上时，M 可以不被阻塞，这避免了大量创建 M 导致的开销。但如果 G 因慢系统调用而阻塞，那么 M 也会一起阻塞，但在阻塞前会与 P 解绑，P 会尝试与其他 M 绑定继续运行其他 G。但若没有现成的 M，Go 运行时会建立新的 M，这也是系统调用可能导致系统线程数量增加的原因


### 无缓冲 channel 的惯用法:
* 用作信号传递(1 对 1 通知信号和 1 对 n 通知)
* 用于替代锁机制

### 带缓冲 channel 的惯用法:
* 用作消息队列
* 用作计数信号量（counting semaphore）


sync 包提供了 Go 语言的一系列低级同步原语。Channel 机制也是建立在低级同步原语的基础之上的，因此也叫做高级同步原语。在一些追求高性能的场景下，直接使用低级同步原语可以获得更好的性能。


读写锁适合应用在具有一定并发量且读多写少的场合

atomic 包更适合一些对性能十分敏感、并发量较大且读多写少的场合。

范型局限:
* Go 接口和结构体不支持泛型方法
* 泛型约束不能作为类型声明
* 泛型约束只能是接口，而不能是结构体

## 编程技巧: 巧用大括号控制变量作用域

```go

var name string
var folder string
var mod string
...
{
   prompt := &survey.Input{
      Message: "请输入目录名称：",
   }
   err := survey.AskOne(prompt, &name)
   if err != nil {
      return err
   }

   ...
}
{
   prompt := &survey.Input{
      Message: "请输入模块名称(go.mod中的module, 默认为文件夹名称)：",
   }
   err := survey.AskOne(prompt, &mod)
   if err != nil {
      return err
   }
   ...
}
{
   // 获取hade的版本
   client := github.NewClient(nil)
   prompt := &survey.Input{
      Message: "请输入版本名称(参考 https://github.com/gohade/hade/releases，默认为最新版本)：",
   }
   err := survey.AskOne(prompt, &version)
   if err != nil {
      return err
   }
   ...
}
```


## TCP Socket 编程模型

基于 TCP 的自定义应用层协议通常有两种常见的定义模式：二进制模式：
* 采用长度字段标识独立数据包的边界。采用这种方式定义的常见协议包括 MQTT（物联网最常用的应用层协议之一）、SMPP（短信网关点对点接口协议）等；
* 文本模式：采用特定分隔符标识流中的数据包的边界，常见的包括 HTTP 协议等。

阻塞 / 非阻塞，是以内核是否等数据全部就绪后，才返回（给发起系统调用的应用线程）来区分的。

* 阻塞 I/O(Blocking I/O)
![image](https://static001.geekbang.org/resource/image/66/70/66e154f76d647a51b45fcfddf4697c70.jpg?wh=1920x1047)
* 非阻塞 I/O（Non-Blocking I/O）

![image](https://static001.geekbang.org/resource/image/4c/b3/4c5e3980f756e03b9ca023185b91b5b3.jpg?wh=1920x1047)
* I/O 多路复用（I/O Multiplexing）
![image](https://static001.geekbang.org/resource/image/4c/b3/4c5e3980f756e03b9ca023185b91b5b3.jpg?wh=1920x1047)

> 在非阻塞模型下，位于用户空间的 I/O 请求发起者通常会通过轮询的方式，去一次次发起 I/O 请求，直到读到所需的数据为止。不过，这样的轮询是对 CPU 计算资源的极大浪费，因此，非阻塞 I/O 模型单独应用于实际生产的比例并不高。

> 虽然目前主流 socket 网络编程模型是 I/O 多路复用模型，但考虑到这个模型在使用时的体验较差，Go 语言将这种复杂性隐藏到运行时层，并结合 Goroutine 的轻量级特性，在用户层提供了基于 I/O 阻塞模型的 Go socket 网络编程模型，这一模型就大大降低了 gopher 在编写 socket 应用程序时的心智负担。

test