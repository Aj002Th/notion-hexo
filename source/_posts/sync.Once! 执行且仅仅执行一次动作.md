---
categories: 源码分析
tags:
  - go
description: ''
permalink: sync-once/
title: 'sync.Once: 执行且仅仅执行一次动作'
cover: /images/676e58f53e170caf75c77817df0c367d.jpg
date: '2024-04-18 19:51:00'
updated: '2024-04-19 19:00:00'
---

# 是什么


sync.Once 是 Go 语言中的一种同步原语，用于确保某个操作或函数在并发环境下只被执行一次。


它只有一个导出的方法，即 Do，该方法接收一个函数参数。在 Do 方法被调用后，该函数将被执行，而且只会执行一次，即使在多个协程同时调用的情况下也是如此。


# 解决了什么问题


对于同一个 sync.Once 实例，可以确保通过调用 Do 执行传入 Do 中的方法，执行且仅执行一次


主要用于以下场景

- 单例模式：确保全局只有一个实例对象，避免重复创建资源
- 延迟初始化：在程序运行过程中需要用到某个资源时，通过 sync.Once 动态地初始化该资源
- 只执行一次的操作：例如只需要执行一次的配置加载、数据清理等操作

# Quick Start


创建和使用


```go
once := sync.Once{}
for i := 0; i < 10; i++ {
	go func() {
		once.Do(func(){
			fmt.Println("hello")
		})
	}()
}
time.Sleep(time.Second * 3)
// hello 只会打印一次
```


# 源码分析


代码量很少很少啦


```go
type Once struct {
	done uint32
	m    Mutex
}

func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```


sync.Once 使用变量 done 来记录函数的执行状态，使用 `sync.Mutex` 和 `sync.atomic` 来保证线程安全的读取 done 

- Do 方法通过原子操作 `sync.atomic` 读取 done，判断函数是否触发过，未触发才继续往下走，调用 doSlow 方法
- doSlow 全局上锁保证同时只有一个 goroutine 能进入，取锁成功后对 done 做第二次校验，避免并发问题。如果 done 的值仍为 0，证明 f 函数没有被执行过，此时执行 f 函数，最后通过原子操作 `atomic.StoreUint32` 将 done 变量的值设置为 1

## 一些 Q&A


### 为什么要使用 atomic 原子操作来对于 done 进行读写


直观的来看，就是 done 变量在 Do 方法中是没有被 sync.Mutex 保护的，如果使用`o.done == 0`和 `o.done = 1`来替代`atomic.LoadUint32(&o.done) == 0` 和 `atomic.StoreUint32(&o.done, 1)` 会导致并发读写冲突，这肯定是不好的


p.s. 类似的竞态冲突都可以通过编译时加入 `-race` 参数被检测出来，这是检查并发问题的好工具


这会导致出现对于 done 的读取产生不确定的结果


在执行原子期间，其他 goroutine 无法修改 o.done 的值，因此不会产生竞争条件


这里其实我感觉如果不用 atomic 原子指令的话其实道理上也许也是能保证程序的正常运行的

1. double check 能确保 `f()`一定只会执行一遍所以能保证功能不出错，所以即使判断出错，也就是有更多的 goroutine 进入 doSlow 在功能层面也没事
2. done 变量就是个布尔值不复杂，读不出什么异常值

但是使用 atomic 确保没有安全问题肯定是更好的选择


### **为什么会单独封装一个 doSlow 方法**


将慢路径（slow-path）代码从 Do 方法中分离出来，使得 Do 方法的快路径（fast-path）函数体很小，能够被内联，从而提高性能


### **为什么要 double check**


第一次检查：避免不必要的锁竞争


第二次检查：确保 `f()` 真的只会执行一次


`atomic.LoadUint32(&o.done) == 0` 为真时，可能会有多个 goroutine 进入 doSlow 方法，我们需要确保 `f()` 执行完毕后，其他 goroutine 拿到锁进入临界区后不要再执行 `f()` 了，所以需要再检查一次 done 变量


### 为什么不能优化成仅使用 atomic + flag 的实现


例如优化成如下代码


```go
type Once struct {
	done uint32
}

func (o *Once) Do(f func()) {
	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
		f()
	}
}
```


`atomic.CompareAndSwapUint32` 这个指令完全同时能完成原子性的取值判断和修改操作，这样实现可以省去一个比较重的 `sync.Mutex` 互斥锁


这是源码注释中提到的一个问题，说是国外的网友问太多了，就专门加了一段注释


```go
// Note: Here is an incorrect implementation of Do:
//
//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
//		f()
//	}
//
// Do guarantees that when it returns, f has finished.
// This implementation would not implement that guarantee:
// given two simultaneous calls, the winner of the cas would
// call f, and the second would return immediately, without
// waiting for the first's call to f to complete.
// This is why the slow path falls back to a mutex, and why
// the atomic.StoreUint32 must be delayed until after f returns.
```


关键在于并发的goroutine在调用 Do 方法时，当 Do 方法返回时，我们期望的是初始化函数 f 要执行完毕，但是这个实现第一个 goroutine 在使用 f 进行初始化时，后续并发的 goroutine 会立即返回，尽管f还没有执行完


这带来的一个问题就是：后续的 goroutine 由于在 f 未执行完就先返回了，在他们的视角里资源是初始化完成了，所以在使用这些未初始化的资源的时候，会出现意想不到的问题，比如 panic 等


所以不能这么简单的实现


### 为什么不在 f() 后面 直接 atomic.StoreUint32(&o.done, 1) 而用 defer


考虑 f() 中出现了 panic 的情况，即使程序在外层 recover 回来了，doSlow 也会因此直接返回了，后续的`atomic.StoreUint32(&o.done, 1)` 在这种情况下就不会得到执行


即使 f() 没有运行成功，也应当认为 f() 已经运行过了


使用 defer 能保证如果程序 recover 回来了，那么`atomic.StoreUint32(&o.done, 1)` 就会得到执行， Once 会说：`f()` 已经运行过了


# 注意事项

- 在 sync.Once 的 Do 方法中重复调用 Do 方法，在首次调用时会导致死锁。因为内外两层 Do 方法都要抢锁，sync.Mutex 又是个不可重入锁，就形成了循环等待
- 如果要传递 sync.Once 变量，要用指针传递而不是值拷贝，不然将 once 值拷贝有可能会导致 once 会重复执行的问题

# 参考资料

- 项目文档：[https://pkg.go.dev/sync#Once](https://pkg.go.dev/sync#Once)
- [Go sync.Once：简约而不简单的并发利器 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/623090559)
- [Go sync.Once的三重门 (colobu.com)](https://colobu.com/2021/05/05/triple-gates-of-sync-Once/)
- [The Go Memory Model - The Go Programming Language](https://go.dev/ref/mem)
- [后端 - go语言happens-before原则及应用 - 个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000039729417)
