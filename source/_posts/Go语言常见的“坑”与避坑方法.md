---
title: Go语言常见的“坑”与避坑方法
date: 2022-11-10 00:05:03
tags: ["Go"]
---
Go语言虽然语法比较简单,但稍有不慎,可能会掉进指定的“坑”里,特别是从其他语言转Go的人来说。本文将介绍Go语言常见的“坑”与避坑方法。

## for循环
### 循环变量的重用
```go
func main() {
	var m = []int{1, 2, 3, 4, 5}
	for i, v := range m {
		go func() {
			time.Sleep(time.Millisecond * 100)
			fmt.Println(i, v)
		}()
	}

	time.Sleep(time.Second * 1)
}
```
以上代码会输出:
```go
4 5
4 5
4 5
4 5
4 5
```
<!-- more -->
而不是预期的(由于goroutine调度不一定按这个顺序):
```
0 1
1 2
2 3
3 4
4 5
```
原因是因为i,v这两个循环变量在 for range 语句中仅会被声明一次，且在每次迭代中都会被重用。即上面代码等价于:
```go

func main() {
    var m = []int{1, 2, 3, 4, 5}  
             
    {
      i, v := 0, 0
        for i, v = range m {
            go func() {
                time.Sleep(time.Millisecond * 100)
                fmt.Println(i, v)
            }()
        }
    }

    time.Sleep(time.Second * 1)
}
```
等到goroutine执行时,for循环已经结束,i,v值分别为4和5

这个问题的一个变种是遍历了一个结构体数组,并在循环里取数组项的地址:
```go
package main

import "fmt"

type user struct {
	name string
}

func main() {
	userList := []user{{name: "n1"}, {name: "n2"}}
	userList2 := make([]*user, 2)
	for i, user := range userList {
		userList2[i] = &user
	}
	for _, user := range userList2 {
		fmt.Println(user.name)
	}
}

```
输出:
```go
n2
n2
```

解决方案:
* 为闭包函数增加参数并在创建goroutine时传入
```go

func main() {
    var m = []int{1, 2, 3, 4, 5}

    for i, v := range m {
        go func(i, v int) {
            time.Sleep(time.Millisecond * 100)
            fmt.Println(i, v)
        }(i, v)
    }

    time.Sleep(time.Second * 1)
}
```
* 巧用作用域,重新声明i,v变量
```go
func main() {
	var m = []int{1, 2, 3, 4, 5}

	for i, v := range m {
		i, v := i, v
		go func() {
			time.Sleep(time.Millisecond * 100)
			fmt.Println(i, v)
		}()
	}

	time.Sleep(time.Second * 1)
}
```


### 参与循环的是 range 表达式的副本
例子:
```go

func main() {
    var a = [5]int{1, 2, 3, 4, 5}
    var r [5]int

    fmt.Println("original a =", a)

    for i, v := range a {
        if i == 0 {
            a[1] = 12
            a[2] = 13
        }
        r[i] = v
    }

    fmt.Println("after for range loop, r =", r)
    fmt.Println("after for range loop, a =", a)
}
```
期望输出:
```go
original a = [1 2 3 4 5]
after for range loop, r = [1 12 13 4 5]
after for range loop, a = [1 12 13 4 5]
```
实际输出:
```go
original a = [1 2 3 4 5]
after for range loop, r = [1 2 3 4 5]
after for range loop, a = [1 12 13 4 5]
```
原因就是参与 for range 循环的是 range 表达式的副本。也就是说，在上面这个例子中，真正参与循环的是 a 的副本，而不是真正的 a。

解决方案:
* 用切片代替数组
```go

func main() {
    var a = [5]int{1, 2, 3, 4, 5}
    var r [5]int

    fmt.Println("original a =", a)

    for i, v := range a[:] {
        if i == 0 {
            a[1] = 12
            a[2] = 13
        }
        r[i] = v
    }

    fmt.Println("after for range loop, r =", r)
    fmt.Println("after for range loop, a =", a)
}
```
用 a[:]替代了原先的 a,原因在于切片副本的结构体中的 array，依旧指向原切片对应的底层数组，所以我们对切片副本的修改也都会反映到底层数组 a 上去

切片内部表示:
![image](https://static001.geekbang.org/resource/image/d1/22/d1dcfdb6fd74c88ca300212d07b04422.jpg?wh=1920x1047)
### 遍历 map 中元素的随机性
对map key,value的for range遍历是无序的

如果我们在循环的过程中，对 map 进行了修改，那么这样修改的结果是否会影响后续迭代呢？这个结果和我们遍历 map 一样，具有随机性。
```go
var m = map[string]int{
    "tony": 21,
    "tom":  22,
    "jim":  23,
}

counter := 0
for k, v := range m {
    if counter == 0 {
        delete(m, "tony")
    }
    counter++
    fmt.Println(k, v)
}
fmt.Println("counter is ", counter)
```
可能输出一:
```go
tony 21
tom 22
jim 23
counter is  3
```
可能输出二:
```go

tom 22
jim 23
counter is  2
```
## switch
### default分支执行
无论 default 分支出现在什么位置，它都只会在所有 case 都没有匹配上的情况下才会被执行
```go
    i := 0
	switch i {
	default:
		fmt.Println("default...")
	case 1:
		fmt.Println("i is 1")
	case 0:
		fmt.Println("i is 0")
	}
```
输出:
```
i is 0
```

### case语句
多个值应该这样写
```go
switch x {
    case 0,1:
        // ...
}
```
而不是:
```go
switch x {
    case 0:
    case 1:
}
```

### 函数表达式

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
> 由于 fallthrough 的存在，Go 不会对 case2 的表达式做求值操作，而会直接执行 case2 对应的代码分支

```

### type switch里是不能fallthrough的
```go
    var i interface{}
	switch i.(type) {
	case int:
		fmt.Println("i is int")
		fallthrough
	case int64:
		fmt.Println("i is int64")
	}
```
如上代码会报错:
```
Cannot use 'fallthrough' in the type switch
```
## 日期
### AddDate
```go
    const df = "2006-01-02"
	srcTime, _ := time.Parse(df, "2022-10-31")
	fmt.Println(srcTime.AddDate(0, 1, 0))
```
输出:
```
2022-12-01 00:00:00 +0000 UTC
```
而不是预期的2022-11-30

AddDate的注释:
```go
// AddDate returns the time corresponding to adding the
// given number of years, months, and days to t.
// For example, AddDate(-1, 2, 3) applied to January 1, 2011
// returns March 4, 2010.
//
// AddDate normalizes its result in the same way that Date does,
// so, for example, adding one month to October 31 yields
// December 1, the normalized form for November 31.
func (t Time) AddDate(years int, months int, days int) Time {
    // ...
}
```
即不管是加年，还是加月, AddDate也是按Date的方式来标准化结果,简单理解就是10月31号加一个月等于11月31号，但11月只有30号，所以结果为12月1号

解决方案:
如果增减之后的日期不合法（当月不存在本日），需要做一些特殊处理,可以参考开源库: [go.timeconv](https://github.com/Andrew-M-C/go.timeconv/blob/master/timeconv.go#L18)
## defer
### defer与return的顺序
```go
package main

import "fmt"

func deferFunc() int {
    fmt.Println("defer func called")
    return 0
}

func returnFunc() int {
    fmt.Println("return func called")
    return 0
}

func returnAndDefer() int {

    defer deferFunc()

    return returnFunc()
}

func main() {
    returnAndDefer()
}
```
输出结果:
```
return func called
defer func called
```
结论:return之后的语句先执行，defer后的语句后执行

再看一个例子:
```go
package main

import "fmt"

func returnButDefer() (t int) {  //t初始化0， 并且作用域为该函数全域

    defer func() {
        t = t * 10
    }()

    return 1
}

func main() {
    fmt.Println(returnButDefer())
}
```
> 该returnButDefer()本应的返回值是1，但是在return之后，又被defer的匿名func函数执行，所以t=t*10被执行，最后returnButDefer()返回给上层main()的结果为10

### defer中包含panic
```go
package main

import (
    "fmt"
)

func main()  {

    defer func() {
       if err := recover(); err != nil{
           fmt.Println(err)
       }else {
           fmt.Println("fatal")
       }
    }()

    defer func() {
        panic("defer panic")
    }()

    panic("panic")
}
```
结果:
```
defer panic
```
panic仅有最后一个可以被revover捕获,defer中若发生panic,会覆盖之前抛出的panic
## 并发
### 共享资源并发访问
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	intSlice := []int{1, 2, 3, 4, 5}
	wg.Add(len(intSlice))
	ans := 0
	for _, v := range intSlice {
		v := v
		go func() {
			defer wg.Done()
			ans += v
		}()
	}
	wg.Wait()
	fmt.Printf("ans:%v", ans)
	return
}
```
由于ans += v并非原子操作,因此结果不一定是15

解决方案: 引入Atomic/Mutext/Channel等同步原语

### 副本问题
```go
import (
	"sync"
	"sync/atomic"
)

func main() {

	var wg sync.WaitGroup

	ans := int64(0)
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go newGoRoutine(wg, &ans)
	}
	wg.Wait()
}

func newGoRoutine(wg sync.WaitGroup, i *int64) {
	defer wg.Done()
	atomic.AddInt64(i, 1)
	return
}

```
结果会死锁,原因是传给newGoRoutine函数的是sync.WaitGroup对象的值,而值传递导致执行wg.Done方法时原始WaitGroup对象相应的state字段不会对应修改

解决方案: 传指针

### 不可重入导致死锁问题
```go
import (
	"fmt"
	"sync"
)

var lock sync.Mutex

func main() {
	wg := sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func(i int) {
			defer wg.Done()
			lock.Lock()
			defer lock.Unlock()
			fmt.Printf("goroutine %v get the lock\n", i)
			callOtherFunc()
		}(i)
	}
	wg.Wait()
}

func callOtherFunc() {
	lock.Lock()
	defer lock.Unlock()
	fmt.Println("callOtherFunc...")
}
```
上述代码会导致死锁,原因是sync.Mutex不是可重入锁(go原生也没提供可重入锁),因此callOtherFunc里再次获取锁会阻塞,最终导致死锁

解决方案: callOtherFunc不再获取锁,或对sync.Mutex作二次封装支持可重入锁
