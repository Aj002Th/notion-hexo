---
categories: 源码分析
tags:
  - go
description: ''
permalink: sync-mutex/
title: sync.Mutex：标准库的互斥锁实现
cover: /images/de91d8193c1b7d27e88f220af42a71b8.jpg
date: '2024-04-18 19:51:00'
updated: '2024-04-19 18:59:00'
---

# 是什么


sync.Mutex 是 go 原生提供的互斥锁实现，也是最基本的同步原语了


合理利用锁即可避免并发编程中由于竞争引发的一些逻辑错误


# Quick Start


sync.Mutex 对外暴露的接口有三个

1. sync.Mutex.Lock 请求锁，如果锁忙，则阻塞。
2. sync.Mutex.TryLock 请求锁，如果锁忙则，则返回 false。这是非阻塞的获取锁的方式。
3. sync.Mutex.Unlock 释放锁

一个简单的例子：


```go
package main

import (
	"sync"
	"time"
)

func main() {
	var m sync.Mutex
	cnt := 0
	for i := 0; i < 10; i++ {
		go func() {
			m.Lock()
			cnt++
			m.Unlock()
		}()
	}

	time.Sleep(time.Second) // 保证所有协程执行完	
	fmt.Println(cnt)
}
```


# 源码分析


## 核心结构


```go
type Mutex struct {
	state int32
	sema  uint32
}

// A Locker represents an object that can be locked and unlocked.
type Locker interface {
	Lock()
	Unlock()
}
```


有一个默认的锁实现 Mutex，而且通过接口提供了扩展性，可以通过实现 Locker 接口，来实现自己的互斥锁。所以使用的时候应该用声明 Locker 接口变量，以后想更改实现就会很容易（虽然一般也不会想着去变）

- Mutex.state：表示当前锁的状态，长度是 32 位
	- 低三位分别表示：锁是空闲还是忙（mutexLocked）、现在阻塞队列头部的 goroutine 正在被唤醒（mutexWoken）、是正常模式还是饥饿模式（mutexStarving）
	- 剩余 29位用于记录当前互斥锁上等待的 goroutine 的数目（waiterCount）

	state 这个字段可以说是很省了，是一个 4 合 1 的字段

- Mutex.sema：控制由于锁而挂起、唤醒的 goroutine 的信号量

## 主流程


### Mutex.Lock


```go
// Lock locks m.
// If the lock is already in use, the calling goroutine
// blocks until the mutex is available.
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```

1. 先用 atomic 原子操作，用 CAS 操作试探抢锁，抢锁成功就直接返回
2. 抢锁失败了进入 lockSlow 方法做进一步处理

这里单独提取了 lockSlow 方法和之前看过的 sync.Once 里面的操作一模一样，就是为了简化 Lock 函数体，Lock 的 fast-path 部分可以内联优化，这个操作在标准库中应用挺多的


### Mutex.lockSlow


这块逻辑挺长的，先看一下这里的关键概念是锁的模式：正常 or 饥饿


正常模式下，新 goroutine 和新唤醒的 goroutine 有同样的资格去自旋抢锁，这里可能出现的问题是新 goroutine 往往是更加活跃的它更可能占据着 cpu 所以取锁的概率会相对更高；为了避免等待队列中的 goroutine 迟迟取不到锁，所以当有 goroutine 等待取锁的时间过长，锁会进入饥饿模式，此时所有 goroutine 按照先来后到的顺序依次进入阻塞队列排队取锁


这里列出源码并添加了详细的注释：


```go
func (m *Mutex) lockSlow() {
	var waitStartTime int64  // 等待时间
	starving := false  // 是否为饥饿模式
	awoke := false  // 阻塞队列队头的 goroutine 是否已经唤醒
	iter := 0  // 自旋次数
	old := m.state  // 临时保存原值

	for {
		// [自旋]
		// Don't spin in starvation mode, ownership is handed off to waiters
		// so we won't be able to acquire the mutex anyway.
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// 当前 goroutine 是被重新唤醒的
			// 设置锁的 mutexWoken 位
			// 避免 unlock 又继续唤醒阻塞队列队头的 goroutine
			// Active spinning makes sense.
			// Try to set mutexWoken flag to inform Unlock
			// to not wake other blocked goroutines.
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}

		// [构造新 state 值]
		new := old
		// 构造新的 mutexLocked
		// 是否要参与抢锁：锁是空闲状态 + 非饥饿模式 才有资格去抢
		// Don't try to acquire starving mutex, new arriving goroutines must queue.
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		// 构造新的 waiterCount
		// 如果当前协程没有资格抢锁, 则等待拿锁的协程数一定+1
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}

		// 正常/饥饿模式的标志位更新
		// The current goroutine switches mutex to starvation mode.
		// But if the mutex is currently unlocked, don't do the switch.
		// Unlock expects that starving mutex has waiters, which will not
		// be true in this case.
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
		// 更新的标记位记录：是否有重新被唤醒的 goroutine 正在运行
		// 清除掉标记, 
		if awoke {
			// The goroutine has been woken from sleep,
			// so we need to reset the flag in either case.
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken
		}

		// [试图更新 state]
		// 基于 old state 构造出的 new state 才能更新 old state
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			// [上锁成功]
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}

			// [准备将协程挂起]
			// 老协程放在 queueLifo 队头, 优先取锁
			// 新协程放在 queueLifo 队尾, 排队取锁
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}

			// [挂起协程, 协程进入阻塞队列等待唤醒]
			// unlock 解锁的时候会唤醒
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)

			// [队头协程被重新唤醒]
			// 等待时间超限就会进入饥饿模式
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			// [上锁成功]
			// 锁已处于饥饿模式, queueLifo 队头协程会直接获得锁, 因为此时不允许自旋的协程竞争
			if old&mutexStarving != 0 {
				// If this goroutine was woken and mutex is in starvation mode,
				// ownership was handed off to us but mutex is in somewhat
				// inconsistent state: mutexLocked is not set and we are still
				// accounted as waiter. Fix that.
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				// 发现自己是 queueLifo 中最后一个协程并且现在处于饥饿模式
				// 则恢复正常模式
				if !starving || old>>mutexWaiterShift == 1 {
					// Exit starvation mode.
					// Critical to do it here and consider wait time.
					// Starvation mode is so inefficient, that two goroutines
					// can go lock-step infinitely once they switch mutex
					// to starvation mode.
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			// 锁处于正常模式, queueLifo 队头协程需要和新协程一起自旋竞争锁
			awoke = true
			iter = 0
		} else {
			// [尝试更新 state 失败, 从头开始重试自旋抢锁]
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}
```


这个 lockSlow 的逻辑还是比较复杂的，大体的逻辑如下：

1. 判断当前情况下能否进入自旋
	- 条件一：正常模式
	- 条件二：runtime_canSpin(iter)返回值为 true ，查看函数实现可以知道需要同时满足
		1. 当前互斥锁处于正常模式
		2. 当前运行的机器是多核CPU，且GOMAXPROCS>1
		3. 至少存在一个其他正在运行的处理器P，并且它的本地运行队列（local runq）为空
		4. 当前goroutine进行自旋的次数小于4

	如果能进入自旋，则通过循环的方式自旋请求锁，否则直接跳过这一阶段

2. 计算互斥锁的新状态：mutexLocked、mutexWoken、mutexStarving
3. 尝试更新互斥锁的状态
如果 2 中计算 state 表明有资格进行抢锁（new state 中 mutexLocked 位是 1），更新这一步就相当于用 CAS 尝试获得锁，如果成功直接返回，失败获取现在 mutex 的状态并回到 1

	如果 2 中计算 state 表面没有资格进行锁争抢（因为锁已经被其他 goroutine 获取或是此时锁已经处于饥饿模式），则会进入阻塞队列进行等待。新 goroutine 会被放入阻塞队列尾部，已经被唤醒过但是还是没抢到锁的 goroutine 会放到阻塞队列的头部


	阻塞队列头部的 goroutine 会因为锁的释放操作而被唤醒，如果唤醒时发现锁已经处于了饥饿模式，那么就能直接获取到锁，如果锁此时处于正常模式，则需要回到 1 中和新 goroutine 一起自旋抢锁


这个部分应该就是 mutex 里最复杂的部分了，总结成流程图如下：


![Untitled.png](/images/6ed05d4f1d4bb6291b68327579e40551.png)


额外总结一下 mutex 的饥饿模式和正常模式：

- 饥饿模式是为了避免大量新的请求互斥锁的 goroutine 的出现导致大量自旋，让进入等待队列的 goroutine 一直抢不到锁引发饥饿的问题
- 正常模式下，新的请求互斥锁的 goroutine 可以通过自旋等待锁的释放，如此可以避免 goroutine 的切换来提高总体的执行效率
- 饥饿模式下，新的请求互斥锁的 goroutine 不允许自旋，直接加入等待队列，按照先来后到取锁
- 等待队列中 goroutine 的唤醒是严格按照 FIFO 来唤醒挂起的
- 进入饥饿模式的条件为：请求锁的等待时间超过 1ms
- 退出饥饿模式的条件为：等待队列为空 或 请求锁的等待时间小于 1ms

### Mutex.Unlock


Mutex.Unlock 方法用于释放锁，主干仍然是十分简单，细节都在 unlockSlow 这个辅助函数里

1. atomic.AddInt32(&m.state, -mutexLocked) 尝试快速解锁
2. 快速解锁失败则调用 m.unlockSlow(new) 慢速解锁

```go
// Unlock unlocks m.
// It is a run-time error if m is not locked on entry to Unlock.
//
// A locked Mutex is not associated with a particular goroutine.
// It is allowed for one goroutine to lock a Mutex and then
// arrange for another goroutine to unlock it.
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
		m.unlockSlow(new)
	}
}
```


### Mutex.unlockSlow


```go
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		fatal("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		// 正常模式的处理方法
		old := new
		for {
			// 如果锁没有waiter,或者锁有其他以下已发生的情况之一，则后面的工作就不用做了，直接返回
      // 1. 锁处于锁定状态，表示锁已经被其他 goroutine 获取了
      // 2. 锁处于被唤醒状态，这表明有阻塞队列队头goroutine被唤醒，不用再尝试唤醒其他goroutine
      // 3. 锁处于饥饿模式，那么锁之后会被直接交给等待队列队头goroutine
			// If there are no waiters or a goroutine has already
			// been woken or grabbed the lock, no need to wake anyone.
			// In starvation mode ownership is directly handed off from unlocking
			// goroutine to the next waiter. We are not part of this chain,
			// since we did not observe mutexStarving when we unlocked the mutex above.
			// So get off the way.
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			
			// 减少等待的 gorouttine 数目并唤醒阻塞队列队头的 goroutine
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// 饥饿模式的处理方法
		// 直接唤醒阻塞队列中的下一个 goroutine
		// Starving mode: handoff mutex ownership to the next waiter, and yield
		// our time slice so that the next waiter can start to run immediately.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```


下面就进入 unlockSlow 部分了,这个相对而言也并不复杂


正常模式：

- 如果没有等待者，或者state低三位不全为0，那就不用唤醒等待队列中的 goroutine
- 如果有等待者，就将锁设置为 unlock 并唤醒阻塞队列队头的 goroutine

饥饿模式：

- 直接唤醒等待队列中的下一个 goroutine 即可

## 其他内容


### Mutex.TryLock


源码 + 注释如下


```go
// TryLock tries to lock m and reports whether it succeeded.
//
// Note that while correct uses of TryLock do exist, they are rare,
// and use of TryLock is often a sign of a deeper problem
// in a particular use of mutexes.
func (m *Mutex) TryLock() bool {
	old := m.state

	// 看看有没有拿锁的条件
	if old&(mutexLocked|mutexStarving) != 0 {
		return false
	}

	// 尝试拿锁
	// There may be a goroutine waiting for the mutex, but we are
	// running now and can try to grab the mutex before that
	// goroutine wakes up.
	if !atomic.CompareAndSwapInt32(&m.state, old, old|mutexLocked) {
		return false
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
	return true
}
```


就是把 Lock 的 slow-path 给去掉了，变成一个非阻塞的方法


# 总结


go 标准库中的 sync.Mutex 实现综合了自旋锁和互斥锁，先尝试自旋，如果能自旋拿到锁，就不用切换 goroutine 了，这样更加高效；如果长时间自旋，就是浪费 cpu 资源，所以多次尝试无果后会将想要拿锁的 goroutine 放到阻塞队列中挂起，当锁被释放时再将位于阻塞队列队头的 goroutine 唤醒


为了避免阻塞队列中挂起的 goroutine 长时间得不到运行，所以锁会有正常模式和饥饿模式。有 goroutine 长时间得不到运行时锁进入饥饿模式，不允许自旋抢锁，所有 goroutine 依据阻塞队列中的次序依次取锁，以这种方式解决饥饿问题


从 sync.Mutex 带有自旋的设计而言，如果我们通过 sync.Mutex 只锁定执行耗时很低的关键代码，例如锁定某个变量的赋值，性能是非常不错的（因为等待锁的goroutine不用被挂起，持有锁的goroutine会很快释放锁）。**所以，我们在使用互斥锁时，应该只锁定真正的临界区**。

