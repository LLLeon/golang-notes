# Mutex 的源码实现

本文基于 go1.11 版本。

## Mutex 使用

在深入源码之前，要先搞清楚一点，对 Golang 中互斥锁 `sync.Mutex` 的操作是程序员的主动行为，可以看作是是一种协议，而不是强制在操作前必须先获取锁。

这样说可能有点抽象，看下面这段代码：

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type People struct {
	mux  sync.Mutex
	Name string
	Age  uint8
}

func (p *People) IncAge() {
	p.mux.Lock()

	time.Sleep(3 * time.Second)
	p.Age++

	p.mux.Unlock()
}

func main() {
	leo := &People{Name: "leo", Age: 18}

	innerIncTime := time.Now().Second()
	fmt.Println("with mutex  inc time:", innerIncTime)
	go leo.IncAge()

	time.Sleep(time.Second)
	outerIncTime := time.Now().Second()
	fmt.Println("without mutex inc time:", outerIncTime)
	leo.Age++
	fmt.Println("without mutex inc result:", leo.Age)

	fmt.Println("mutex status:", leo.mux)
	time.Sleep(2 * time.Second)
	fmt.Println("with mutex inc result:", leo.Age)
	fmt.Println("Two seconds later mutex status:", leo.mux)
}
```

在执行 `leo.Age++` 之前已经加锁了，如果是需要强制获取锁的话，这里会等待 3 秒直到锁释放后才能执行，而这里没有获取锁就可以直接对 Age 字段进行操作，输出结果：

```bash
with mutex  inc time: 19
without mutex inc time: 20
without mutex inc result: 19
mutex status: {1 0}
with mutex inc result: 20
Two seconds later mutex status: {0 0}
```

所以，如果在一个 goroutine 中对锁执行了 Lock()，在另一个 goroutine 可以不用理会这个锁，直接进行操作（当然不建议这么做）。

还有一点需要注意的是，锁只和具体变量关联，与特定 goroutine 无关。虽然可以在一个 goroutine 中加锁，在另一个 goroutine 中解锁（如通过指针传递变量，或全局变量都可以），但还是建议在同一个代码块中进行成对的加锁解锁操作。

## 源码分析

这是 Mutex 的 [源码链接](https://github.com/golang/go/blob/master/src/sync/mutex.go) 。

### Mutex 结构

Mutex 表示一个互斥锁，其零值就是未加锁状态的 mutex，无需初始化。在首次使用后不要做值拷贝，这样可能会使锁失效。

```go
type Mutex struct {
	state int32  // 表示锁当前的状态
	sema  uint32 // 信号量，用于向处于 Gwaitting 的 G 发送信号
}
```
### 几个常量

这里用位操作来表示锁的不同状态。

```go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota

	starvationThresholdNs = 1e6
)
```

####mutexLocked

值为 1，第一位为 1，表示 mutex 已经被加锁。根据 `mutex.state & mutexLocked` 的结果来判断 mutex 的状态：该位为 1 表示已加锁，0 表示未加锁。

#### mutexWoken

值为 2，第二位为 1，表示 mutex 是否被唤醒。根据 `mutex.state & mutexWoken` 的结果判断 mutex 是否被唤醒：该位为 1 表示已被唤醒，0 表示未被唤醒。

#### mutexStarving

值为 4，第三位为 1，表示 mutex 是否处于饥饿模式。根据 `mutex.state & mutexWoken` 的结果判断 mutex 是否处于饥饿模式：该位为 1 表示处于饥饿模式，0 表示正常模式。

#### mutexWaiterShift

值为 3，表示 `mutex.state` 右移 3 位后即为等待的 `goroutine` 的数量。

#### starvationThresholdNs

值为 1000000 纳秒，即 1ms，表示将 mutex 切换到饥饿模式的等待时间阈值。这个常量在源码中有大篇幅的注释，理解这段注释对理解程序逻辑至关重要，翻译整理如下：

引入这个常量是为了保证出现 mutex 竞争情况时的公平性。mutex 有两种操作模式：**正常模式和饥饿模式**。

正常模式下，等待者以 FIFO 的顺序排队来获取锁，但被唤醒的等待者发现并没有获取到 mutex，并且还要与新到达的 goroutine 们竞争 mutex 的所有权。新到达的 goroutine 们有一个优势 —— 它们已经运行在 CPU 上且可能数量很多，所以一个醒来的等待者有很大可能会获取不到锁。在这种情况下它处在等待队列的前面。如果一个 goroutine 等待 mutex 释放的时间超过 1ms，它就会将 mutex 切换到饥饿模式。

在饥饿模式下，mutex 的所有权直接从对 mutex 执行解锁的 goroutine 传递给等待队列前面的等待者。新到达的 goroutine 们不要尝试去获取 mutex，即使它看起来是在解锁状态，也不要试图自旋（等也白等，在饥饿模式下是不会给你的），而是自己乖乖到等待队列的尾部排队去。

如果一个等待者获得 mutex 的所有权，并且看到以下两种情况中的任一种：1) 它是等待队列中的最后一个，或者 2) 它等待的时间少于 1ms，它便将 mutex 切换回正常操作模式。

正常模式有更好地性能，因为一个 goroutine 可以连续获得好几次 mutex，即使有阻塞的等待者。而饥饿模式可以有效防止出现位于等待队列尾部的等待者一直无法获取到 mutex 的情况。

### 自旋锁操作

在开始看 Mutex 源码前要先介绍几个与自旋锁相关的函数，源码中通过这几个函数实现了对自旋锁的操作。这几个函数实际执行的代码都是在 runtime 包中实现的。

#### runtime_canSpin

代码[具体位置](https://github.com/golang/go/blob/master/src/runtime/proc.go)。

由于 Mutex 的特性，自旋需要比较保守的进行，原因参考上面 `starvationThresholdNs` 常量的注释。

限制条件是：只能自旋少于 4 次，而且仅当运行在多核机器上并且 GOMAXPROCS>1；最少有一个其它正在运行的 P，并且本地的运行队列 runq 里没有 G 在等待。与 runtime mutex 相反，不做被动自旋，因为可以在全局 runq 上或其它 P 上工作。

```go
// Active spinning for sync.Mutex.
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
	// sync.Mutex is cooperative, so we are conservative with spinning.
	// Spin only few times and only if running on a multicore machine and
	// GOMAXPROCS>1 and there is at least one other running P and local runq is empty.
	// As opposed to runtime mutex we don't do passive spinning here,
	// because there can be work on global runq or on other Ps.
	if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}
```

#### **runtime_doSpin**

代码具体位置同上。

执行自旋操作，这个函数是用汇编实现的，函数内部循环调用 PAUSE 指令。PAUSE 指令什么都不做，但是会消耗 CPU 时间。

```go
/go:linkname sync_runtime_doSpin sync.runtime_doSpin
//go:nosplit
func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}
```

#### runtime_SemacquireMutex

代码[具体位置](https://github.com/golang/go/blob/master/src/runtime/sema.go)。发送获取到 Mutex 的信号。

```go
//go:linkname sync_runtime_SemacquireMutex sync.runtime_SemacquireMutex
func sync_runtime_SemacquireMutex(addr *uint32, lifo bool) {
	semacquire1(addr, lifo, semaBlockProfile|semaMutexProfile)
}
```

#### **runtime_Semrelease**

代码具体位置同上。发送释放 Mutex 的信号。

```go
//go:linkname sync_runtime_Semrelease sync.runtime_Semrelease
func sync_runtime_Semrelease(addr *uint32, handoff bool) {
	semrelease1(addr, handoff)
}
```

### Lock

这是 Lock 方法的全部源码，先看一下整体逻辑，下面会分段解释：

首先是直接调用 CAS 尝试获取锁，如果获取到则将锁的状态从 0 切换为 1 并返回。获取不到就进入 for 循环，通过自旋来等待锁被其它 goroutine 释放，只有两个地方 break 退出 for 循环而获取到锁。

#### 源码实现分析

刚进入函数，会尝试获取锁：

```go
if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
```

直接通过调用 `CompareAndSwapInt32` 这个方法来检查锁的状态是否是 0，如果是则表示可以加锁，将其状态转换为 1，当前 goroutine 加锁成功，函数返回。

`CompareAndSwapInt32` 这个方法是汇编实现的，在单核CPU上运行是可以保证原子性的，但在多核 CPU 上运行时，需要加上 LOCK 前缀来对总线加锁，从而保证了该指令的原子性。

至于 `race.Enabled` ，这里是判断是否启用了竞争检测，即程序编译或运行时是否加上了 `-race` 子命令。关于竞争检测如想深入了解可以看官方博客，见参考目录 1。在这里可以不用理会。

如果 mutex 已经被其它 goroutine 持有，则进入下面的逻辑。先定义了几个变量：

```go
var waitStartTime int64 // 当前 goroutine 开始等待的时间
starving := false       // mutex 当前的所处的模式
awoke := false          // 当前 goroutine 是否被唤醒
iter := 0               // 自旋迭代的次数
old := m.state          // 保存 mutex 当前状态
```

进入 for 循环后，先检查是否可以进行自旋：

如上所述，**不要在饥饿模式下进行自旋**，因为在饥饿模式下只有等待者们可以获得 mutex 的所有权，这时自旋是不可能获取到锁的。

能进入执行自旋逻辑部分的条件：当前不是饥饿模式，而且当前还可以进行自旋（见上面的 runtime_canSpin 函数）。

然后是判断能否唤醒当前 goroutine 的四个条件：根据 1）`!awoke` 和 2）`old&mutexWoken == 0` 来判断当前 goroutine 还没有被唤醒；3）`old>>mutexWaiterShift != 0` 表示还有其它在等待的 goroutine；4）如果当前 goroutine 状态还没有变，就将其状态切换为 `old|mutexWoken`， 即唤醒状态 。

```go
// old&(mutexLocked|mutexStarving) == mutexLocked 表示 mutex 当前不处于饥饿模式。
// 即 old & 0101 == 0001，old 的第一位必定为 1，第三位必定为 0，即未处于饥饿模式。
if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
    // 这时自旋是有意义的，通过把 mutexWoken 标识为 true，以通知 Unlock 方法就先不要叫醒其它
    // 阻塞着的 goroutine 了。
	if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
		atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
		awoke = true
	}
    // 将当前 goroutine 标识为唤醒状态后，执行自旋操作，计数器加一，将当前状态记录到 old，继续循环等待
	runtime_doSpin()
	iter++
	old = m.state
	continue
}
```

如果不能进行自旋操作，进入下面的逻辑：

如果 mutex 当前处于正常模式，将 new 的第一位即锁位设置为 1；如果 mutex 当前已经被加锁或处于饥饿模式，则当前 goroutine 进入等待队列；如果 mutex 当前处于饥饿模式，而且 mutex 已被加锁，则将 new 的第三位即饥饿模式位设置为 1。

```go
new := old
// 不要尝试获取处于饥饿模式的锁，新到达的 goroutine 们必须排队。
if old&mutexStarving == 0 {
	new |= mutexLocked
}
// 没有获取到锁，当前 goroutine 进入等待队列。
// old & 0101 != 0，那么 old 的第一位和第三位至少有一个为 1，即 mutex 已加锁或处于饥饿模式。
if old&(mutexLocked|mutexStarving) != 0 {
	new += 1 << mutexWaiterShift
}
// 当前 goroutine 将 mutex 切换到饥饿模式。但如果当前 mutex 是解锁状态，不要切换。
// Unlock 期望处于饥饿模式的 mutex 有等待者，在这种情况下不会这样。
if starving && old&mutexLocked != 0 {
	new |= mutexStarving
}
```

设置好 new 后，继续下面的逻辑：

当 goroutine 被唤醒时，如果 new 还没有被唤醒，则发生了不一致的 mutex 状态，抛出错误；否则就重置 new 的第二位即唤醒位为 0。

```go
if awoke {
	if new&mutexWoken == 0 {
		throw("sync: inconsistent mutex state")
	}
	new &^= mutexWoken
}
```

接下来会调用 CAS 来将 mutex 当前状态由 old 更新为 new：

如更新成功，`old&(mutexLocked|mutexStarving) == 0` 表示 mutex 未锁定且未处于饥饿模式，则 break 跳出循环，当前 goroutine 获取到锁。

如果当前的 goroutine 之前已经在排队了，就排到队列的前面。`runtime_SemacquireMutex(&m.sema, queueLifo)` 这个函数就是做插队操作的，如果 queueLifo == true，就把当前 goroutine 插入到等待队列的前面。

继续往下，如果 mutex 当前是处于饥饿模式，则修改等待的 goroutine 数量和第三位即饥饿模式位，break 跳出循环，当前 goroutine 获取到锁；如果是正常模式，继续循环。

```go
if atomic.CompareAndSwapInt32(&m.state, old, new) {
    // old & 0101 == 0，old 的第一位和第三位必定不是 1，即 mutex 未锁定且未处于饥饿模式。
	if old&(mutexLocked|mutexStarving) == 0 {
		break // locked the mutex with CAS
	}
  
	// If we were already waiting before, queue at the front of the queue.
	queueLifo := waitStartTime != 0
	if waitStartTime == 0 {
        // 之前没排过队，开始计时。
		waitStartTime = runtime_nanotime()
	}
	runtime_SemacquireMutex(&m.sema, queueLifo)
  
    // 确定 mutex 当前所处模式
	starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
	old = m.state
	if old&mutexStarving != 0 {
		// If this goroutine was woken and mutex is in starvation mode,
		// ownership was handed off to us but mutex is in somewhat
		// inconsistent state: mutexLocked is not set and we are still
		// accounted as waiter. Fix that.
		if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
			throw("sync: inconsistent mutex state")
		}
		delta := int32(mutexLocked - 1<<mutexWaiterShift)
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
    // 不是饥饿模式，继续循环
	awoke = true
	iter = 0
} else {
	old = m.state
}
```

### Unlock

Unlock 的源码比较短：

```go
func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
            // 如果没有等待者了，或者一个 goroutine 已经被唤醒，再或者已经获取到锁，无需叫醒任意一个。
            // 在饥饿模式中，所有权直接从执行解锁的 goroutine 传递给下一个等待者。
            // 我们不是这个链的一部分，因为当我们解锁上面的互斥锁时，没有观察到 mutexStarving，
            // 还是乖乖返回吧。
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}

            // 唤醒某个 goroutine
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
                // 释放锁，发送释放信号
				runtime_Semrelease(&m.sema, false)
				return
			}
			old = m.state
		}
	} else {
        // 饥饿模式：将 mutex 所有权传递给下个等待者。
        // 注意：mutexLocked 没有设置，等待者将在醒来后设置它。
        // 但是如果设置了 mutexStarving，仍然认为 mutex 是锁定的，所以新来的 goroutine 不会获取到它。
		runtime_Semrelease(&m.sema, true)
	}
}

```




## 参考

1. [Data Race Detector](https://golang.org/doc/articles/race_detector.html)
2. [Golang 互斥锁内部实现](https://zhuanlan.zhihu.com/p/27608263)
3. [信号量，锁和 golang 相关源码分析](http://www.gaufung.com/post/ji-zhu/semaphore_lock_and_something_else)