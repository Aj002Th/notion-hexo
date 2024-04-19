---
categories: 源码分析
tags:
  - go
  - 并发
description: ''
permalink: ants/
title: 'ants: 协程池源码学习'
cover: /images/24c3bef21dbe219801f421672961ae73.jpg
date: '2023-12-08 13:07:00'
updated: '2024-04-19 21:50:00'
---

# 是什么


`ants`是一个高性能的 goroutine 池，实现了对大规模 goroutine 的调度管理、goroutine 复用，允许使用者在开发并发程序的时候限制 goroutine 数量，复用资源，达到更高效执行任务的效果。


# 解决了什么问题

- 提升性能：主要面向一类场景，大批量轻量级并发任务，任务执行成本与协程创建/销毁成本量级接近
- 并发资源控制：研发能够明确系统全局并发度以及各个模块的并发度上限
- 协程生命周期控制：实时查看当前全局并发的协程数量；有一个统一的紧急入口释放全局协程

带来的价值：

1. 限制并发的 goroutine 数量
2. 复用 goroutine，减轻 runtime 调度压力，提升程序性能
3. 规避过多的 goroutine 侵占系统资源（CPU&内存）

# Quick Start


创建 goroutine 池


```go
pool, _ := ants.NewPool(100)
```


提交任务


```go
ants.Submit(func(){
	// do something
})
```


动态调整容量


```go
pool.Tune(1000) // Tune its capacity to 1000
pool.Tune(100000) // Tune its capacity to 100000
```


释放资源


```go
pool.Release()
```


# 源码分析


## 核心结构


架构图中包含了所有核心结构，并展现了分层关系，开发中最为常用的还是 Pool，主要沿着 Pool 这条线进行研究，PoolWithFunc 实现逻辑其实和 Pool 大差不差


![Untitled.png](/images/355436df9ba90da6c1684082e2ea8815.png)


### pool


```go
// Pool accepts the tasks and process them concurrently,
// it limits the total of goroutines to a given number by recycling goroutines.
type Pool struct {
	// capacity of the pool, a negative value means that the capacity of pool is limitless, an infinite pool is used to
	// avoid potential issue of endless blocking caused by nested usage of a pool: submitting a task to pool
	// which submits a new task to the same pool.
	capacity int32

	// running is the number of the currently running goroutines.
	running int32

	// lock for protecting the worker queue.
	lock sync.Locker

	// workers is a slice that store the available workers.
	workers workerQueue

	// state is used to notice the pool to closed itself.
	state int32

	// cond for waiting to get an idle worker.
	cond *sync.Cond

	// workerCache speeds up the obtainment of a usable worker in function:retrieveWorker.
	workerCache sync.Pool

	// waiting is the number of goroutines already been blocked on pool.Submit(), protected by pool.lock
	waiting int32

	purgeDone int32
	stopPurge context.CancelFunc

	ticktockDone int32
	stopTicktock context.CancelFunc

	now atomic.Value

	options *Options
}
```


Pool 就是所谓的协程池，管理 goroutine，限制 goroutine 数目

- capacity：协程池容量，最大 goroutine 数量限制
- running：正在运行的协程数目
- lock：自制的轻量自旋锁，用于保护 goWorker 队列
- workers：goWorker 队列，真正意义上的协程池，存放着可以复用的 goroutine 资源
- state：协程池状态标识，0-打开；1-关闭
- cond：go中的信号量，是一种 goroutine 同步工具。如果 pool 配置 blocking 模式，则会用于协调 goroutine 的唤醒和挂起
- workerCache：goWorker 对象池，缓存着被释放的 goWorker，用于 goWorker 的资源复用
- waiting：被阻塞的协程数目
- purgeDone：标识释放 goWorker 的协程是否关闭
- stopPurge：用于关闭释放 goWorker 的协程
- ticktockDone：标识时间更新协程是否关闭
- stopTicktock：用于关闭时间更新协程的函数
- now：缓存当前时间，在 goWorker 执行完任务放回 goWorker 队列时需要记录任务完成时间，通过缓存+异步更新的方式，减少 time.Now() 的调用次数
- optons：一些定制化配置

### workerQueue


```go
type workerQueue interface {
	len() int
	isEmpty() bool
	insert(worker) error
	detach() worker
	refresh(duration time.Duration) []worker // clean up the stale workers and return them
	reset()
}
```


workerQueue 是一个接口，定义了数据操作所需的 api


接口的实现有 `workerStack` 和 `loopQueue` ，从命名即可看出一个是栈结构一个是循环队列的结构


如果 `Pool`使用了 PreAlloc 容量预分配配置，会选用 `loopQueue` 实现，否则会默认选用`workerStack` 的实现


这里所谓的是否预分配其实指的是他们内部实现中存放 `goWorker`结构的 slice 容量是否预分配


`loopQueue` 的容量在创建后就不能够更改了，而`workerStack` 可以通过 Tune() 方法动态更新容量


继续关注一下 `workerStack` 的实现


### workerStack


```go
type workerStack struct {
	items  []worker
	expiry []worker
}
```

- items：协程池中可用的 goWorker
- expiry：已过期需要清理释放的 goWorker

### goWorker


```go
// goWorker is the actual executor who runs the tasks,
// it starts a goroutine that accepts tasks and
// performs function calls.
type goWorker struct {
	// pool who owns this worker.
	pool *Pool

	// task is a job should be done.
	task chan func()

	// lastUsed will be updated when putting a worker back into queue.
	lastUsed time.Time
}
```


goWorker 是任务真正的执行者，可以认为它代表了协程池中的协程资源

- pool：goWorker 所属的协程池
- task：goWorker 用于接收异步任务的管道
- lastUsed：goWorker 回收到协程池的时间（上一次结束任务运行的时间）

## 主流程


![Untitled.png](/images/aaa1b050dd6116ece7c74917d3e4f570.png)


### NewPool 创建协程池


```go
// NewPool instantiates a Pool with customized options.
func NewPool(size int, options ...Option) (*Pool, error) {
	// 校验用户入参，设置 options
	if size <= 0 {
		size = -1
	}

	opts := loadOptions(options...)

	if !opts.DisablePurge {
		if expiry := opts.ExpiryDuration; expiry < 0 {
			return nil, ErrInvalidPoolExpiry
		} else if expiry == 0 {
			opts.ExpiryDuration = DefaultCleanIntervalTime
		}
	}

	if opts.Logger == nil {
		opts.Logger = defaultLogger
	}

	// 构造 Pool 数据结构
	p := &Pool{
		capacity: int32(size),
		lock:     syncx.NewSpinLock(),
		options:  opts,
	}
	// 构造 goWorker 对象池
	p.workerCache.New = func() interface{} {
		return &goWorker{
			pool: p,
			task: make(chan func(), workerChanCap),
		}
	}
	// 构造 goWorker 队列，依据是否预分配注入不同的实现结构
	if p.options.PreAlloc {
		if size == -1 {
			return nil, ErrInvalidPreAllocSize
		}
		p.workers = newWorkerQueue(queueTypeLoopQueue, size)
	} else {
		p.workers = newWorkerQueue(queueTypeStack, 0)
	}

	// 构造用于并发控制的 sync.Cond
	p.cond = sync.NewCond(p.lock)

	// 启动负责释放 goWorker 协程
	p.goPurge()
	// 启动负责更新 now 的协程
	p.goTicktock()

	return p, nil
}
```


NewPool 方法用于创建一个新的协程池

1. 校验用户入参，设置 options
2. 构造 Pool 数据结构
3. 构造 goWorker 对象池
4. 构造 goWorker 队列，依据是否预分配注入不同的实现结构
5. 构造用于并发控制的 sync.Cond
6. 启动负责释放 goWorker 协程
7. 启动负责更新 now 的协程

### Pool.Submit 提交任务


```go
// Submit submits a task to this pool.
//
// Note that you are allowed to call Pool.Submit() from the current Pool.Submit(),
// but what calls for special attention is that you will get blocked with the last
// Pool.Submit() call once the current Pool runs out of its capacity, and to avoid this,
// you should instantiate a Pool with ants.WithNonblocking(true).
func (p *Pool) Submit(task func()) error {
	if p.IsClosed() {
		return ErrPoolClosed
	}

	w, err := p.retrieveWorker()
	if w != nil {
		w.inputFunc(task)
	}
	return err
}
```


Pool.Submit 方法的作用是将任务提交给协程池执行

1. 检查协程池是否已经关闭，如果关闭则直接返回 err
2. 调用 retrieveWorker 从协程池中获取一个可供使用的 goWorker
3. 通过 channel 将任务发送给 goWorker 执行

### Pool.retrieveWorker 获取可用的goWorker


```go
// retrieveWorker returns an available worker to run the tasks.
func (p *Pool) retrieveWorker() (w worker, err error) {
	p.lock.Lock()

retry:
	// First try to fetch the worker from the queue.
	if w = p.workers.detach(); w != nil {
		p.lock.Unlock()
		return
	}

	// If the worker queue is empty, and we don't run out of the pool capacity,
	// then just spawn a new worker goroutine.
	if capacity := p.Cap(); capacity == -1 || capacity > p.Running() {
		p.lock.Unlock()
		w = p.workerCache.Get().(*goWorker)
		w.run()
		return
	}

	// Bail out early if it's in nonblocking mode or the number of pending callers reaches the maximum limit value.
	if p.options.Nonblocking || (p.options.MaxBlockingTasks != 0 && p.Waiting() >= p.options.MaxBlockingTasks) {
		p.lock.Unlock()
		return nil, ErrPoolOverload
	}

	// Otherwise, we'll have to keep them blocked and wait for at least one worker to be put back into pool.
	p.addWaiting(1)
	p.cond.Wait() // block and wait for an available worker
	p.addWaiting(-1)

	if p.IsClosed() {
		p.lock.Unlock()
		return nil, ErrPoolClosed
	}

	goto retry
}
```


Pool.retrieveWorker 将获取一个空闲的 goWorker 用于任务的执行

1. 要操作 workerQueue，所以需要上锁保护
2. 调用 workerQueue.detach() 尝试从 workerQueue 中取出一个 goWorker 返回
3. 检查当前正在使用的 goWorker 是否到达协程池的容量，如果容量未超出限额，则通过 workerCache 对象池获取并启动一个 goWorker 返回
4. 如果容量超出限额，且 Pool 为非阻塞模式，则返回 err
5. 如果容量超出限额，且 Pool 为阻塞模式但等待执行的任务数目到达配置的上限，返回 err
6. 如果容量超出限额，且 Pool 为阻塞模式但等待执行的任务数目未达上线，则通过 cond 挂起

### workerQueue.detach 取出一个 goWorker


这里就关注一下 `workerStack` 的实现，`loopQueue` 也是类似的


```go
func (wq *workerStack) detach() worker {
	l := wq.len()
	if l == 0 {
		return nil
	}

	w := wq.items[l-1]
	wq.items[l-1] = nil // avoid memory leaks
	wq.items = wq.items[:l-1]

	return w
}
```


workerQueue.detach 做的工作其实就是从 slice 中取出最后一个元素，比较简单


值得注意的点是 `wq.items[l-1] = nil` 这步防止内存泄漏的操作


详细可参考 [issue: Avoid memory leak #107](https://github.com/panjf2000/ants/pull/107)


### goWorker.run 执行任务


```go
// run starts a goroutine to repeat the process
// that performs the function calls.
func (w *goWorker) run() {
	w.pool.addRunning(1)
	go func() {
		defer func() {
			w.pool.addRunning(-1)
			w.pool.workerCache.Put(w)
			if p := recover(); p != nil {
				if ph := w.pool.options.PanicHandler; ph != nil {
					ph(p)
				} else {
					w.pool.options.Logger.Printf("worker exits from panic: %v\n%s\n", p, debug.Stack())
				}
			}
			// Call Signal() here in case there are goroutines waiting for available workers.
			w.pool.cond.Signal()
		}()

		for f := range w.task {
			if f == nil {
				return
			}
			f()
			if ok := w.pool.revertWorker(w); !ok {
				return
			}
		}
	}()
}
```


阶段一：goWorker 的运行

- 不断从 task channel 中获取任务并执行，任务执行完毕后通过 revertWorker 将 goWorker 结构放回 workerQueue，goroutine 并没有退出而是阻塞等待新任务
- 这里就是对于 goroutine 的复用，也是 ants 高性能的关键

阶段二：goWorker 的释放

- 当前 goWorker 在负责释放 goWorker 协程中被调用 finish 方法，finish 触发 goWorker 销毁的方法是往 task 中注入空值，从 task 中取得 nil，由此进入释放流程
- goWorker 在执行过程中出现 panic，或是回归 workerQueue 失败（例如 Pool 已关闭），也会被 defer 兜底，并进入释放流程
- 将 goWorker 放入 WorkerCache 对象池
- 通过 cond.Signal() 唤醒一个阻塞等待的协程获取 goWorker 启动任务

### Pool.revertWorker 回收goWorker


```go
// revertWorker puts a worker back into free pool, recycling the goroutines.
func (p *Pool) revertWorker(worker *goWorker) bool {
	if capacity := p.Cap(); (capacity > 0 && p.Running() > capacity) || p.IsClosed() {
		p.cond.Broadcast()
		return false
	}

	worker.lastUsed = p.nowTime()

	p.lock.Lock()
	// To avoid memory leaks, add a double check in the lock scope.
	// Issue: https://github.com/panjf2000/ants/issues/113
	if p.IsClosed() {
		p.lock.Unlock()
		return false
	}
	if err := p.workers.insert(worker); err != nil {
		p.lock.Unlock()
		return false
	}
	// Notify the invoker stuck in 'retrieveWorker()' of there is an available worker in the worker queue.
	p.cond.Signal()
	p.lock.Unlock()

	return true
}
```


Pool.revertWorker 负责将 goWorker 放回到 workerQueue 中

- 更新 goWorker 记录的时间 lastUsed，lastUsed 是用于判断闲置的 goWorker 是否应该被回收的依据
- 将 goWorker 放回 workerQueue 中
- 唤醒一个等待的协程取用 goWorker

### Pool.goPurge 释放goWorker


```go
func (p *Pool) goPurge() {
	if p.options.DisablePurge {
		return
	}

	// Start a goroutine to clean up expired workers periodically.
	var ctx context.Context
	ctx, p.stopPurge = context.WithCancel(context.Background())
	go p.purgeStaleWorkers(ctx)
}
```


回头看在 Pool.New 中使用 Pool.goPurge 启动的用于释放 goWorker 的协程

- 通过 context.Context 来控制协程的退出
- 使用 Pool.purgeStaleWorkers 启动真正的 goWorker 释放协程

```go
// purgeStaleWorkers clears stale workers periodically, it runs in an individual goroutine, as a scavenger.
func (p *Pool) purgeStaleWorkers(ctx context.Context) {
	ticker := time.NewTicker(p.options.ExpiryDuration)

	defer func() {
		ticker.Stop()
		atomic.StoreInt32(&p.purgeDone, 1)
	}()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
		}

		if p.IsClosed() {
			break
		}

		var isDormant bool
		p.lock.Lock()
		staleWorkers := p.workers.refresh(p.options.ExpiryDuration)
		n := p.Running()
		isDormant = n == 0 || n == len(staleWorkers)
		p.lock.Unlock()

		// Notify obsolete workers to stop.
		// This notification must be outside the p.lock, since w.task
		// may be blocking and may consume a lot of time if many workers
		// are located on non-local CPUs.
		for i := range staleWorkers {
			staleWorkers[i].finish()
			staleWorkers[i] = nil
		}

		// There might be a situation where all workers have been cleaned up (no worker is running),
		// while some invokers still are stuck in p.cond.Wait(), then we need to awake those invokers.
		if isDormant && p.Waiting() > 0 {
			p.cond.Broadcast()
		}
	}
}
```


Pool.purgeStaleWorkers 负责 goWorker 的释放工作

- 定时检查 workerQueue 中的 goWorker 是否闲置过长时间，对过期的 goWorker 进行销毁释放
- 对于 goWorker 过期的判断是基于 goWorker.lastUsed 来做的，workerStack 和 loopQueue 这两种 workerQueue 实现中都是通过二分查找的方式来获取过期 goWorker

### Pool.Release 释放协程池


```go
// Release closes this pool and releases the worker queue.
func (p *Pool) Release() {
	if !atomic.CompareAndSwapInt32(&p.state, OPENED, CLOSED) {
		return
	}

	if p.stopPurge != nil {
		p.stopPurge()
		p.stopPurge = nil
	}
	p.stopTicktock()
	p.stopTicktock = nil

	p.lock.Lock()
	p.workers.reset()
	p.lock.Unlock()
	// There might be some callers waiting in retrieveWorker(), so we need to wake them up to prevent
	// those callers blocking infinitely.
	p.cond.Broadcast()
}
```


Pool.Release 在协程池不再使用时，用于销毁，释放系统资源

1. 通过 atomic 包的原子操作，无锁更新 Pool 的状态，标记为关闭
2. 通过 context 关闭辅助协程
3. 释放 workerQueue 的资源
4. 为了防止有 goroutine 还阻塞在`p.cond.Wait()`上，执行一次`p.cond.Broadcast()`

## 其他内容


### PoolWithFunc


和 Pool 的设计整体上都大差不差


只是 Pool 使用 goWorker 来代表 goroutine 资源，需要传入 func 作为任务执行；而PoolWithFunc 使用 goWorkerWithFunc 来代表 goroutine 资源，所需执行的任务函数在协程池创建时就已经指定，提交任务是需要传入的内容就不是是函数了，而是函数运行的参数


### MultiPool


MultiPool 内容会初始化多个协程池，可以根据预先定义的负载均衡策略：轮询或者最少使用策略，从多个协程池中获取 goWorker


Pool 中有保护 workerQueue 的自旋锁，之所以新加了这个，是为了降低锁的颗粒度，以此提高性能


### MultiPoolWithFunc


与 MultiPool 之间的关系就和 Pool 和 PoolWithFunc 之间的关系差不多


### spinLock 自制的轻量级自旋锁


嫌弃 sync.Mutex 太重了，利用 atomic.CompareAndSwapUint32() 这个原子操作实现了一个自旋锁。


与其他类型的锁不同，自旋锁在加锁失败之后不会立刻进入等待，而是会继续尝试。这对于很快就能获得锁的应用来说能极大提升性能，因为能减少加锁和解锁导致的线程切换


加锁操作中使用了指数退避的算法，连续的抢锁失败会让惩罚增加，runtime.Gosched() 会让当前 goroutine 让出 cpu，减少空转


```go
type spinLock uint32

const maxBackoff = 16

func (sl *spinLock) Lock() {
	backoff := 1
	for !atomic.CompareAndSwapUint32((*uint32)(sl), 0, 1) {
		// Leverage the exponential backoff algorithm, see https://en.wikipedia.org/wiki/Exponential_backoff.
		for i := 0; i < backoff; i++ {
			runtime.Gosched()
		}
		if backoff < maxBackoff {
			backoff <<= 1
		}
	}
}

func (sl *spinLock) Unlock() {
	atomic.StoreUint32((*uint32)(sl), 0)
}
```


# 注意事项


> 💡 使用 ants 的一个理由是通过池子来自动管理 goroutine 的生命周期，不用手动管理  
> 考虑一下提交给协程池的任务中又启动了新协程的情景，读了源码自然能够知道，在这种情况下，这些协程是不受 ants 限制的，还是需要手动管理生命周期。所以避免在提交的任务中启动新协程，或是设计好这些新协程的生命周期管理


	考虑一下提交给协程池的任务中又启动了新协程的情景，读了源码自然能够知道，在这种情况下，这些协程是不受 ants 限制的，还是需要手动管理生命周期。所以避免在提交的任务中启动新协程，或是设计好这些新协程的生命周期管理


> 💡 ants 擅长处理的任务应当是耗时短但是量大的小任务  
> 对于耗时长和量小的任务而言，体现不出 ants 对于 goroutine 复用的性能提升


	对于耗时长和量小的任务而言，体现不出 ants 对于 goroutine 复用的性能提升


# 参考资料

- 项目地址：[panjf2000/ants: 🐜🐜🐜 ants is a high-performance and low-cost goroutine pool in Go. (github.com)](https://github.com/panjf2000/ants)
- [GMP 并发调度器深度解析之手撸一个高性能 goroutine pool - Strike Freedom (taohuawu.club)](https://taohuawu.club/archives/high-performance-implementation-of-goroutine-pool)
- [Visually Understanding Worker Pool | by David | Coinmonks | Medium](https://medium.com/coinmonks/visually-understanding-worker-pool-48a83b7fc1f5)
- [Golang 协程池 Ants 实现原理 (qq.com)](https://mp.weixin.qq.com/s/Uctu_uKHk5oY0EtSZGUvsA)
- [Go 每日一库之 ants（源码赏析） - 大俊的博客 (darjun.github.io)](https://darjun.github.io/2021/06/04/godailylib/ants-src/)
