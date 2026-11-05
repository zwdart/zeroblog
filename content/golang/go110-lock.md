---
title: 'Goroutine Lock'
date: '2026-11-05T11:30:30+08:00'
categories: [golang,Goroutine]
tags: [Mutex,RWMutex]
description: ''
draft: false

cover:
  image: 'images/cover.svg'
  alt: '山高月小，水落石出。'
  caption: '山水'
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
---


# Go 中与 Goroutine 相关的锁与同步机制详解

从**"goroutine 视角"**系统梳理 Go 提供的锁/同步原语，每个都配可运行示例，并说清楚它们和 GMP 调度器的交互。

---

## 一、总览：Go 有哪些锁与同步原语

| 原语 | 包 | 一句话定位 | 阻塞时 goroutine 状态 |
|---|---|---|---|
| `Mutex` | `sync` | 互斥锁，最基础 | `_Gwaiting`，让出 P |
| `RWMutex` | `sync` | 读写分离，读并发 | 同上 |
| `Cond` | `sync` | 条件变量，配合 Mutex 等待条件 | `_Gwaiting` |
| `Once` | `sync` | 只执行一次 | 首次执行时其他 G 等待 |
| `WaitGroup` | `sync` | 等一组 goroutine 完成 | `_Gwaiting` |
| `Map` | `sync` | 并发安全 map（读优化） | 内部用锁/原子 |
| `atomic` | `sync/atomic` | 无锁原子操作 | **不阻塞**，CPU 原语 |
| `channel` | 内置 | 通信 + 同步 | `_Gwaiting` |

**核心认知**：除 `atomic` 外，所有锁原语在竞争时都会让当前 goroutine 进入 `_Gwaiting`，**让出 P**，M 去跑别的 G。**阻塞在锁上的 goroutine 不占线程**——这是 Go 能开几十万 goroutine 的共同基础。

---

## 二、Mutex：最基础的互斥

### 系统要点（承接上一轮）

- 无竞争：一次 CAS，约 10ns
- 竞争：先自旋（≤4 次，条件苛刻），失败则 park 到信号量
- 等待 > 1ms：切饥饿模式，直接移交锁，保尾延迟
- **阻塞时 goroutine 让出 P**，M 不空转

### 可运行示例：竞争条件与修复

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	// ❌ 无锁：竞态
	var counter int
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter++ // 读-改-写非原子
		}()
	}
	wg.Wait()
	fmt.Println("无锁结果（不确定，通常 < 1000）:", counter)

	// ✅ 加锁：正确
	var mu sync.Mutex
	counter = 0
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			mu.Lock()
			defer mu.Unlock()
			counter++
		}()
	}
	wg.Wait()
	fmt.Println("加锁结果（必定 1000）:", counter)
}
```

**运行**：`go run main.go`。第一个数字每次跑都可能不同，第二个永远是 1000。

**验证竞态**：`go run -race main.go` 会在第一个循环上报 `DATA RACE`。

---

## 三、RWMutex：读并发，但别默认它更快

### 系统要点

- 多读并发，写独占
- **写者优先**：有写者等待时，新读者被阻塞，防止写者饿死
- 内部维护 `readerCount`、`readerWait`、两套信号量，**读操作开销高于 Mutex 的单次 CAS**
- **读临界区很短时，RWMutex 反而更慢**——簿记成本 > 并发收益

### 可运行示例：对比 Mutex 与 RWMutex

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

var data = make(map[int]int)

func benchMutex(reads, writes int) time.Duration {
	var mu sync.Mutex
	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < writes; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			mu.Lock()
			data[n] = n
			mu.Unlock()
		}(i)
	}
	for i := 0; i < reads; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			mu.Lock()
			_ = data[n%writes]
			mu.Unlock()
		}(i)
	}
	wg.Wait()
	return time.Since(start)
}

func benchRWMutex(reads, writes int) time.Duration {
	var mu sync.RWMutex
	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < writes; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			mu.Lock()
			data[n] = n
			mu.Unlock()
		}(i)
	}
	for i := 0; i < reads; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			mu.RLock()
			_ = data[n%writes]
			mu.RUnlock()
		}(i)
	}
	wg.Wait()
	return time.Since(start)
}

func main() {
	reads, writes := 10000, 100
	fmt.Println("Mutex   :", benchMutex(reads, writes))
	fmt.Println("RWMutex :", benchRWMutex(reads, writes))
	fmt.Println("注意：读临界区极短时，两者差距不大甚至 Mutex 更快；")
	fmt.Println("      读临界区做大（如 sleep 1μs）后，RWMutex 才明显胜出。")
}
```

**关键结论**：跑多次会发现**短临界区下 RWMutex 未必更快**。RWMutex 的收益只在**读临界区足够长（微秒级以上）+ 读远多于写**时才体现。**不要因为"读多写少"就盲目上 RWMutex，先 profile。**

### RWMutex 的两个致命陷阱

```go
// ❌ 陷阱 1：持有读锁时请求写锁 → 死锁（不支持锁升级）
mu.RLock()
mu.Lock()      // 永久阻塞：写锁要求无读者，而自己正持有读锁

// ❌ 陷阱 2：递归读锁 → 写者等待时可能死锁
mu.RLock()
mu.RLock()     // 表面可重入，但若有写者在等，第二次 RLock 会阻塞
               // 而写者又在等第一个 RLock 释放 → 死锁
```

**Go 不支持锁升级，也不支持读锁重入的安全保证。**

---

## 四、Cond：条件变量，配合 Mutex 等条件

### 系统要点

- `sync.Cond` **不是锁**，是"等待某个条件成立"的机制，必须配一把 `Mutex` 使用
- `Wait()` 做三件事：**释放锁 → 阻塞 → 被唤醒后重新加锁**
- `Signal()` 唤醒一个等待者，`Broadcast()` 唤醒全部
- 等待方必须**用 `for` 循环检查条件**，因为唤醒不保证条件成立（虚假唤醒 + 竞争）

### 可运行示例：用 Cond 实现有界队列

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type BoundedQueue struct {
	mu       sync.Mutex
	notEmpty *sync.Cond
	notFull  *sync.Cond
	buf      []int
	cap      int
}

func NewBoundedQueue(cap int) *BoundedQueue {
	q := &BoundedQueue{cap: cap}
	q.notEmpty = sync.NewCond(&q.mu)
	q.notFull = sync.NewCond(&q.mu)
	return q
}

func (q *BoundedQueue) Put(v int) {
	q.mu.Lock()
	defer q.mu.Unlock()
	for len(q.buf) == q.cap { // 必须 for，不能 if
		q.notFull.Wait() // 释放锁、阻塞、醒来重新加锁
	}
	q.buf = append(q.buf, v)
	q.notEmpty.Signal()
}

func (q *BoundedQueue) Get() int {
	q.mu.Lock()
	defer q.mu.Unlock()
	for len(q.buf) == 0 {
		q.notEmpty.Wait()
	}
	v := q.buf[0]
	q.buf = q.buf[1:]
	q.notFull.Signal()
	return v
}

func main() {
	q := NewBoundedQueue(3)

	// 生产者
	go func() {
		for i := 0; i < 10; i++ {
			q.Put(i)
			fmt.Println("放入:", i)
			time.Sleep(100 * time.Millisecond)
		}
	}()

	// 消费者
	for i := 0; i < 10; i++ {
		v := q.Get()
		fmt.Println("取出:", v)
		time.Sleep(200 * time.Millisecond)
	}
	fmt.Println("完成")
}
```

**为什么用 Cond 而不是 channel**：Cond 适合"**同一个锁保护多个条件**"的场景（这里 `notEmpty` 和 `notFull` 共享 `mu`）。channel 做有界队列更简单，但 Cond 能表达更复杂的条件组合（比如"队列非空 且 未关闭"）。**能用 channel 就别用 Cond**——Cond 容易写错（漏 for、漏 Signal、Broadcast 惊群）。

### Cond 的三个硬规则

1. **`Wait()` 必须放在 `for` 循环里**：`for !condition { c.Wait() }`
2. **改条件必须在持锁状态下**，`Signal/Broadcast` 也建议在持锁时调用（避免唤醒后条件又被改回）
3. **`Wait()` 返回时锁一定已重新持有**，所以 `defer Unlock` 依然有效

---

## 五、Once：只执行一次

### 系统要点

- `Once.Do(f)` 保证 `f` 在**所有并发调用中只执行一次**
- 底层用 `atomic.Load` 快路径 + `Mutex` 慢路径：**已执行后，后续调用仅一次原子读，无锁**
- 常用于单例、全局初始化、配置加载

### 可运行示例：并发安全的单例

```go
package main

import (
	"fmt"
	"sync"
)

type Config struct {
	DSN string
}

var (
	instance *Config
	once     sync.Once
)

func GetConfig() *Config {
	once.Do(func() {
		fmt.Println(">>> 初始化 Config（只会打印一次）")
		instance = &Config{DSN: "postgres://localhost"}
	})
	return instance
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			cfg := GetConfig()
			fmt.Printf("goroutine %d 拿到 Config: %p\n", id, cfg)
		}(i)
	}
	wg.Wait()
}
```

**预期输出**：`>>> 初始化 Config` 只出现一次，5 个 goroutine 拿到**同一个指针**。

### Once 的坑

```go
// ❌ 如果 f panic，Once 会认为"已执行"，后续永远不再执行
once.Do(func() { panic("boom") })
once.Do(func() { fmt.Println("永远不执行") }) // 不会打印
```

**`Once.Do` 的语义是"最多执行一次"**，panic 也算"执行过"。需要重试的场景不要用 Once。

---

## 六、sync.Map：不是"并发 map 的银弹"

### 系统要点

- `sync.Map` 针对**读多写少、键集合稳定**的场景优化（内部 `read` 只读 map + `dirty` 可写 map，读走原子无锁）
- **大多数场景下，普通 `map + Mutex` 更快**
- 不适合：频繁增删、写多读少、需要遍历一致性

### 可运行示例：对比 sync.Map 与 map+Mutex

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func benchSyncMap() time.Duration {
	var m sync.Map
	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < 100000; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			m.Store(n, n)
			_, _ = m.Load(n)
		}(i)
	}
	wg.Wait()
	return time.Since(start)
}

func benchMapMutex() time.Duration {
	var mu sync.Mutex
	m := make(map[int]int)
	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < 100000; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			mu.Lock()
			m[n] = n
			_ = m[n]
			mu.Unlock()
		}(i)
	}
	wg.Wait()
	return time.Since(start)
}

func main() {
	fmt.Println("sync.Map     :", benchSyncMap())
	fmt.Println("map + Mutex  :", benchMapMutex())
	fmt.Println("结论：写多读少时 map+Mutex 往往更快；")
	fmt.Println("      sync.Map 适合「读远多于写、键基本不变」的场景（如缓存）。")
}
```

**结论**：`sync.Map` 只在**读极多、写极少、键稳定**时优于 `map+Mutex`。**默认用 `map + RWMutex` 或 `map + Mutex`，除非 profile 证明 sync.Map 更好。**

---

## 七、atomic：不阻塞的无锁原语

### 系统要点

- `sync/atomic` 提供 CPU 级原子指令（CAS、Load、Store、Add、Swap）
- **永不阻塞 goroutine**，不涉及 park/unpark
- 适合：计数器、标志位、指针替换
- `atomic.Value` 可原子替换任意类型，但有"必须存同一具体类型"的限制

### 可运行示例：原子计数器 vs 锁

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var (
		mu      sync.Mutex
		locked  int64
		atomicV int64
		wg      sync.WaitGroup
	)

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			mu.Lock()
			locked++
			mu.Unlock()

			atomic.AddInt64(&atomicV, 1)
		}()
	}
	wg.Wait()
	fmt.Println("Mutex 计数:", locked)
	fmt.Println("Atomic 计数:", atomicV)
}
```

### atomic.Value：原子替换配置

```go
package main

import (
	"fmt"
	"sync/atomic"
	"time"
)

type Config struct {
	Timeout int
}

func main() {
	var av atomic.Value
	av.Store(&Config{Timeout: 1})

	// 读者
	go func() {
		for i := 0; i < 5; i++ {
			cfg := av.Load().(*Config)
			fmt.Println("读到 Timeout:", cfg.Timeout)
			time.Sleep(200 * time.Millisecond)
		}
	}()

	// 写者：原子替换整个指针
	time.Sleep(500 * time.Millisecond)
	av.Store(&Config{Timeout: 99})
	time.Sleep(time.Second)
}
```

**关键**：`atomic.Value` 只能存**同一个具体类型**的值（这里都是 `*Config`），混存不同类型会 panic。**配置热更新用它最合适**——读无锁、写一次替换。

---

## 八、锁与 GMP 的系统交互：阻塞时到底发生了什么

这是"goroutine 锁系统"最核心的部分。

### 8.1 加锁失败 → goroutine park → 让出 P

```
goroutine G1 调用 mu.Lock()
  ├─ 快路径 CAS 失败
  ├─ lockSlow：自旋几次仍失败
  ├─ 将 G1 加入 Mutex 的等待队列
  ├─ runtime_Semacquire(&m.sema)
  │    └─ gopark() → G1 状态 _Gwaiting
  │    └─ 让出 P，M 去跑其他 _Grunnable 的 G
  └─ （锁释放时被唤醒 → _Grunnable → 重新竞争或直接获锁）

关键：阻塞在锁上的 G1 不占 OS 线程，不占 P。
      1000 个 goroutine 等同一把锁，也只多占 1000 份栈（各 2KB 起）。
```

### 8.2 自旋与 P 的关系

Mutex 的自旋条件之一是 **"当前 P 的本地队列为空"**。含义是：**只有当这个 P 上没别的活可干时，才允许空转等锁**。如果 P 队列里还有一堆 G 等着，自旋就是浪费——runtime 会让当前 G 尽快 park，把 P 让给队列里的 G。

**这就是"锁的调度开销"的来源**：竞争激烈时，大量 G 在 park/unpark 之间反复，产生上下文切换成本。**锁竞争越激烈，调度器开销越大**。

### 8.3 观测锁竞争：mutex profile

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"runtime/pprof"
	"sync"
	"time"
)

func main() {
	runtime.SetMutexProfileFraction(1) // 采集所有锁竞争事件
	defer pprof.Lookup("mutex").WriteTo(os.Stdout, 1)

	var mu sync.Mutex
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 1000; j++ {
				mu.Lock()
				time.Sleep(time.Microsecond) // 拉长临界区，制造竞争
				mu.Unlock()
			}
		}()
	}
	wg.Wait()
	fmt.Println("--- 以上是 mutex profile ---")
}
```

运行后输出会包含 `contentions`（竞争次数）和 `delay`（累计等待时间），**按调用栈聚合**——直接告诉你哪把锁、哪一行是竞争热点。

**生产环境**：`http://localhost:6060/debug/pprof/mutex` 直接看。

---

## 九、死锁：锁与锁、锁与 channel 的纠缠

### 9.1 经典死锁：两个 goroutine 交叉加锁

```go
package main

import "sync"

func main() {
	var a, b sync.Mutex

	go func() { a.Lock(); b.Lock(); b.Unlock(); a.Unlock() }()
	go func() { b.Lock(); a.Lock(); a.Unlock(); b.Unlock() }()

	select {} // 等死
}
```

**运行结果**：`fatal error: all goroutines are asleep - deadlock!`（如果两个 goroutine 恰好交叉成功）。

**修复**：**全局固定加锁顺序**（如按地址/ID 排序），或用 `TryLock` 打破循环等待。

### 9.2 死锁检测的边界（承接上一轮）

`checkdead` 只在**所有 M 都闲着、无 cgo/syscall 阻塞、无 timer** 时才判定死锁。**有 cgo 调用卡住时，runtime 会保守地不报死锁，程序静默挂起**。

### 9.3 锁 + channel 混用导致的隐蔽死锁

```go
mu.Lock()
select {
case v := <-ch:   // 持锁等 channel
    // 如果 ch 的发送者也需要 mu → 死锁
}
mu.Unlock()
```

**原则：不要在持锁期间做可能阻塞的操作**（channel 收发、IO、sleep、调用未知函数）。临界区只放必须原子完成的内存操作。

---

## 十、常见坑清单

| 坑 | 后果 | 修复 |
|---|---|---|
| 含 `Mutex` 的结构体按值传递 | 复制锁，互斥失效 | 传指针；CI 开 `go vet` |
| `Unlock` 未加锁的 Mutex | `fatal error: sync: unlock of unlocked mutex` | `defer mu.Unlock()` 紧跟 `Lock` |
| 重复 `Unlock` | 同上 panic | 配对；用 `defer` |
| 同一 goroutine 重入 `Lock` | 永久阻塞 | 重构为 `fooLocked` 内部函数 |
| 持 `RLock` 时 `Lock` | 死锁 | 不支持锁升级，先 `RUnlock` 再 `Lock` |
| `Cond.Wait` 不用 `for` | 条件不成立时被唤醒 → 逻辑错误 | `for !cond { Wait() }` |
| 持锁做 IO / channel / sleep | 临界区拉长，吞吐骤降 | 临界区只做内存操作 |
| `Once.Do` 里 panic | 后续永远不再执行 | 需要重试就别用 Once |
| 用 `sync.Map` 做写多场景 | 比 `map+Mutex` 慢 | profile 后再选 |
| 锁粒度过细 | 多次加解锁 + 不变式被破坏 | 让临界区覆盖完整的不变式 |

---

## 十一、锁 vs channel：怎么选

| 场景 | 推荐 |
|---|---|
| 保护简单共享状态（计数器、map、结构体字段） | `Mutex` / `RWMutex` |
| 传递数据所有权、生产者-消费者 | `channel` |
| 等待多个条件成立（同一锁下多个谓词） | `Cond` |
| 一次性初始化 | `Once` |
| 等一组 goroutine 完成 | `WaitGroup` |
| 高频计数器、标志位 | `atomic` |
| 读极多写极少的缓存 | `sync.Map`（profile 后再选） |

**Rob Pike 的原话**："Don't communicate by sharing memory; share memory by communicating."——但**这不意味着 channel 万能**。简单状态保护用锁更直接、更快、更易读。**为了"Go 风格"把 `Mutex` 能解决的问题硬套 channel，是反模式**（上一轮讲过，用 channel 当锁慢一个数量级）。

---

## 十二、总结

**系统层面三句话**：

1. **除 atomic 外，所有锁原语在竞争时都让 goroutine 进 `_Gwaiting`、让出 P**——阻塞代价是"一份栈"，不是"一个线程"。
2. **Mutex 的竞争策略是"自旋 → park → 1ms 后切饥饿模式"**，在吞吐与尾延迟之间动态权衡；RWMutex 在此之上加读者计数，写者优先。
3. **锁竞争的观测靠 `pprof mutex`**，不要猜；**锁的 bug 靠 `go vet` + `-race` + 代码审查**，不要靠运行。

**使用层面三条纪律**：

1. **`defer Unlock` 紧跟 `Lock`**，绝不手写配对。
2. **含锁结构体传指针**，CI 必开 `go vet`。
3. **临界区只放必须原子完成的内存操作**，不持锁做 IO / channel / sleep。

**选择层面一条原则**：

> **简单共享状态用锁，数据传递用 channel，多条件等待用 Cond，一次性初始化用 Once，高频计数用 atomic。没有"更 Go"的原语，只有"更适合场景"的原语。**

## 十三、RWMutex的两个致命陷阱

这两个“陷阱”并非设计缺陷，而是为了解决一个更根本的并发问题——**写者饥饿（Writer Starvation）**——而主动做出的设计取舍。

`RWMutex`的核心目标是：**在读多写少的场景下，允许多个读者真正并行，从而提升整体吞吐量**，这是单纯的`Mutex`无法做到的。但“允许多个读者”这个能力本身，会直接导致写操作可能永远得不到执行。这两个陷阱，正是为了在开放读并发的同时，确保写操作不会被无限期搁置而付出的“安全代价”。

### 🎯 核心矛盾：写者饥饿

如果只允许读者共享锁，而不做任何限制，就会产生饥饿问题：当读操作持续不断时，写操作可能永远无法获得锁。因为只要还有一个读者持有锁，写者就必须等待，而新的读者又会不断加入，导致写者被“无限续杯”的读者们永久阻塞。

为了解决这个问题，`RWMutex`必须引入**写者优先（Write-preferring）** 策略。

### 🔧 解决方案：写者优先策略

Go的`RWMutex`在源码中实现了严格的写者优先逻辑，核心规则是：**一旦有写者在等待，就会阻止新的读者获取锁**。

这是通过一个名为`readerCount`的原子计数器实现的：
*   **正常情况下**，`readerCount`为正数，代表当前活跃读者的数量。
*   **当写者调用`Lock()`时**，它会将`readerCount`减去一个极大的常数（`rwmutexMaxReaders`），使其**瞬间变为一个很大的负数**。
*   **此时，任何新的`RLock()`调用**都会发现`readerCount`是负数，从而明白“有写者在等待”，并立即**阻塞**，不再获取读锁。

这个机制有效地防止了写者饥饿，但也直接导致了你提到的两个“陷阱”。

### ⚖️ 陷阱背后的权衡：为什么必须禁止这些操作

现在可以理解，这两个限制是为了维护“写者优先”这一核心原则而付出的必要代价。

*   **关于递归读锁（`RLock`嵌套）**：如果一个goroutine在已持有读锁的情况下再次调用`RLock()`，而此时恰好有写者在等待（`readerCount`为负），那么这个goroutine的**第二次`RLock()`调用将会永久阻塞**。由于它无法释放第一次持有的读锁，会导致**自身死锁**，并且该goroutine将永久地阻塞写者，使得整个锁彻底失效。因此，禁止递归读锁是保证“写者优先”策略能够生效的必要条件。

*   **关于锁升级（`RLock` -> `Lock`）**：一个goroutine在持有读锁时尝试获取写锁（`Lock()`），会**立即造成自我死锁**。因为`Lock()`需要等待所有读者（包括它自己）释放读锁，而它自己又在等待`Lock()`返回，这是一个无法解开的循环等待。这个限制并非Go独有，而是读写锁模型的普遍特性，目的是防止破坏互斥的基本规则。

### 💎 总结

`RWMutex`的设计并非完美，而是**在“读并发带来的高吞吐”和“写操作的公平性”之间，选择了后者**。

*   **它要解决的问题**：`Mutex`在读多写少场景下，读操作完全串行，浪费了并发潜力。
*   **它付出的代价**：引入了比`Mutex`更复杂的内部状态，并且必须禁止一些在`Mutex`上语义模糊但不会立即死锁的操作（如递归锁）。
*   **它的适用边界**：只有当**读临界区足够长（微秒级以上）且读操作远多于写操作**时，`RWMutex`的并发收益才能覆盖其内部的簿记开销。否则，简单的`Mutex`往往是更优的选择。