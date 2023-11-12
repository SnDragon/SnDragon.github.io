---
title: Go实现可重入锁
date: 2023-11-12 10:41:52
tags: ["Go", "锁"]
---
Go语言原生支持的Lock(sync.Mutex)不是可重入的，同个协程重复获得锁会panic
```go
package sync

import (
	"sync"
	"testing"
)

func TestLock(t *testing.T) {
	lock := sync.Mutex{}
	lock.Lock()
	t.Log("get lock 1")
	lock.Lock()
	t.Log("get lock 2")
	lock.Unlock()
}
```
```
=== RUN   TestLock
    lock_test.go:11: get lock 1
fatal error: all goroutines are asleep - deadlock!
```
如何实现类似Java语言提供的ReentrantLock,支持可重入呢?
<!-- more -->

### 最简单的版本
主要思路:
1. 保存当前拥有锁的协程id和重入次数
2. 获取锁时，判断锁的拥有者是否为当前协程，是的话说明当前协程已经拥有锁，不需要再调用Lock方法，只需增加重入次数，否则尝试调用Lock方法
3. 释放锁时，若锁的拥有者不是当前协程，则报错。接着把重入次数减1，若结果为0，说明为当前协程最后一个释放锁的操作，调用UnLock方法
```go
package sync

import (
	"fmt"
	"github.com/petermattis/goid"
	"sync"
	"sync/atomic"
)

type ReentrantLock struct {
	m         sync.Mutex
	owner     int64 // lock所属的goroutine id
	recursion int   // 重入次数
}

func NewReentrantLock() *ReentrantLock {
	return &ReentrantLock{}
}

func (r *ReentrantLock) Lock() {
	gid := goid.Get()
	if atomic.LoadInt64(&r.owner) == gid {
		// 当前持有锁的goroutine
		r.recursion++
		return
	}
	r.m.Lock()
	atomic.StoreInt64(&r.owner, gid)
	r.recursion = 1
}

func (r *ReentrantLock) Unlock() {
	gid := goid.Get()
	if gid != atomic.LoadInt64(&r.owner) {
		panic(fmt.Sprintf("wrong the owner(%d): %d!", r.owner, gid))
	}
	r.recursion--
	if r.recursion > 0 {
		return
	}
	// 释放锁
	atomic.StoreInt64(&r.owner, -1)
	r.m.Unlock()
}

```

```go
package sync

import (
	"fmt"
	"sync"
	"testing"
)

func TestReentrantLock_Lock(t *testing.T) {
	c := NewReentrantLock()
	wg := sync.WaitGroup{}
	numGoroutine := 2
	wg.Add(numGoroutine)
	for i := 0; i < numGoroutine; i++ {
		go func(idx int) {
			defer wg.Done()
			c.Lock()
			fmt.Printf("goroutine %v get lock\n", idx)
			c.Lock()
			fmt.Printf("goroutine %v get lock\n", idx)
			c.Unlock()
			c.Unlock()
			fmt.Printf("goroutine %v release lock\n", idx)
		}(i)
	}
	wg.Wait()
	fmt.Println("main goroutine exit")
}
```
运行结果:
```
=== RUN   TestReentrantLock_Lock
goroutine 1 get lock
goroutine 1 get lock
goroutine 1 release lock
goroutine 0 get lock
goroutine 0 get lock
goroutine 0 release lock
main goroutine exit
--- PASS: TestReentrantLock_Lock (0.00s)
```