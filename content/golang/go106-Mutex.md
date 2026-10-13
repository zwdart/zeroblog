---
title: 'Goroutine sync.Mutex 互斥锁详解'
date: '2026-10-13T09:49:26+08:00'
categories: ['golang']
tags: ['goroutine','Mutex']
description: 'Mutex是互斥锁，用来保护共享资源，防止多个 goroutine 同时读写导致数据竞争'
draft: false

cover:
  image: 'images/cover.svg'
  alt: '山高月小，水落石出。'
  caption: '山水'
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
---

# Go 语言 `sync.Mutex` 互斥锁详解

`sync.Mutex` 是 Go 标准库 `sync` 包提供的**互斥锁**，用来保护共享资源，防止多个 goroutine 同时读写导致数据竞争。

---

## 一、为什么需要互斥锁？

先看一个没有锁的经典问题：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			counter++   // ⚠️ 数据竞争
		}()
	}

	wg.Wait()
	fmt.Println("counter =", counter)
}
```

多次运行，输出经常**小于 1000**，比如 987、993、1000 不固定。原因：

- `counter++` 不是原子操作，实际是三步：读值 → 加一 → 写回
- 多个 goroutine 同时执行时，可能两个都读到 5，都写回 6，结果少加了一次

用 `go run -race main.go` 会明确报出 `DATA RACE`。

**加锁就能解决：**

```go
var mu sync.Mutex

go func() {
	defer wg.Done()
	mu.Lock()
	counter++
	mu.Unlock()
}()
```

多次运行都稳定输出 `counter = 1000`。

---

## 二、基本用法

`sync.Mutex` 只有两个方法：

| 方法 | 作用 |
| :--- | :--- |
| `Lock()` | 加锁，若已被其他 goroutine 持有则阻塞等待 |
| `Unlock()` | 解锁，唤醒一个正在等待的 goroutine |

**必须成对出现**，`Lock` 后一定要 `Unlock`，否则其他 goroutine 会永久阻塞（死锁）。

**推荐写法：`defer` 解锁**

```go
mu.Lock()
defer mu.Unlock()
// ... 临界区代码
```

即使临界区里 `panic` 或 `return`，`defer` 也能保证解锁，避免死锁。

**`sync.Mutex` 零值可用**，不需要显式初始化：

```go
var mu sync.Mutex   // ✅ 直接可用
mu.Lock()
// ...
mu.Unlock()
```

---

## 三、典型示例：并发安全的计数器

```go
package main

import (
	"fmt"
	"sync"
)

type SafeCounter struct {
	mu sync.Mutex
	n  int
}

func (c *SafeCounter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}

func (c *SafeCounter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.n
}

func main() {
	c := &SafeCounter{}
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			c.Inc()
		}()
	}

	wg.Wait()
	fmt.Println("counter =", c.Value())   // 稳定输出 1000
}
```

**要点**：

- 锁和它保护的数据放在同一个结构体里，是 Go 的惯用法
- 方法内部加锁，调用者无需关心锁的存在
- 通过指针接收者调用，避免复制结构体（也就避免了复制锁）

---

## 四、回到你的项目：日志写入

你的 `internal/logger/logger.go` 里已经用到了这个模式：

```go
var (
	logDir string
	mu     sync.Mutex
)

func Write(entry string) error {
	mu.Lock()
	defer mu.Unlock()

	filename := fmt.Sprintf("http_%s.log", time.Now().Format("2006-01-02"))
	path := filepath.Join(logDir, filename)

	f, err := os.OpenFile(path, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		return fmt.Errorf("打开日志文件失败: %w", err)
	}
	defer f.Close()

	_, err = f.WriteString(entry)
	return err
}
```

为什么这里需要锁？

- 如果你以后改成**并发调用多个 LLM**（比如同时跑 5 个 Prompt 做对比），多个 goroutine 会同时调用 `logger.Write`
- 没有锁的话，两条日志内容可能在文件里**交错**，比如：

```
[请求 URL] https://api.openai.com/v1/[请求方法] POST
```

- 加锁后，一次只允许一个 goroutine 写文件，日志内容完整

**注意**：`os.File` 的 `Write` 本身在 Linux 上是原子的，但我们的日志入口做了"打开文件 → 写入 → 关闭"三步，中间可能被打断，所以锁是必要的。

---

## 五、常见错误

### 错误 1：忘记解锁

```go
mu.Lock()
if err != nil {
	return err   // ❌ 忘了 Unlock，其他 goroutine 永远阻塞
}
mu.Unlock()
```

**修正**：始终用 `defer mu.Unlock()` 紧跟 `Lock()`。

```go
mu.Lock()
defer mu.Unlock()
if err != nil {
	return err
}
```

### 错误 2：复制包含锁的结构体

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c Counter) Inc() {   // ❌ 值接收者，复制了锁
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

值接收者会**复制整个结构体**，包括 `Mutex`。复制后的锁是另一个锁，保护不了原来的数据。

**修正**：用指针接收者。

```go
func (c *Counter) Inc() {   // ✅ 指针接收者
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

### 错误 3：重入死锁

`sync.Mutex` **不可重入**。同一个 goroutine 已经持有锁，再次 `Lock` 会永久阻塞。

```go
func (c *SafeCounter) A() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.B()   // ❌ B 里又 Lock，死锁
}

func (c *SafeCounter) B() {
	c.mu.Lock()   // 阻塞
	defer c.mu.Unlock()
	// ...
}
```

**修正**：拆分内部方法，不要在持有锁的情况下调用会再次加锁的方法。

```go
func (c *SafeCounter) A() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.bLocked()   // ✅ 内部方法，不再加锁
}

func (c *SafeCounter) bLocked() {
	// 不加锁，假定调用者已持有锁
}
```

### 错误 4：锁的粒度过大

```go
mu.Lock()
defer mu.Unlock()

data := fetchFromNetwork()     // ❌ 网络请求耗时 2 秒
cache[key] = data
```

锁里做耗时操作（网络、磁盘、大计算），会让其他 goroutine 长时间等待，性能急剧下降。

**修正**：只在**真正操作共享数据**时加锁。

```go
data := fetchFromNetwork()     // ✅ 网络请求不加锁

mu.Lock()
cache[key] = data
mu.Unlock()
```

---

## 六、进阶：`sync.RWMutex` 读写锁

如果读多写少，用 `sync.RWMutex` 性能更好：

```go
type Cache struct {
	mu sync.RWMutex
	m  map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
	c.mu.RLock()          // 读锁，多个 goroutine 可同时持有
	defer c.mu.RUnlock()
	v, ok := c.m[key]
	return v, ok
}

func (c *Cache) Set(key, value string) {
	c.mu.Lock()           // 写锁，独占
	defer c.mu.Unlock()
	c.m[key] = value
}
```

| 锁类型 | 方法 | 并发性 |
| :--- | :--- | :--- |
| 读锁 | `RLock` / `RUnlock` | 多个读可并行 |
| 写锁 | `Lock` / `Unlock` | 独占，读写互斥 |

**使用场景**：
- **读多写少**（如缓存）→ `RWMutex`
- **读写均衡或写多** → `Mutex`

不要为了"看起来更快"而盲目用 `RWMutex`，它的内部实现比 `Mutex` 复杂，写多时反而更慢。

---

## 七、进阶：`defer` 的代价

`defer` 有轻微开销。如果在**极端热路径**（每秒调用数百万次），可以手动解锁：

```go
mu.Lock()
c.n++
mu.Unlock()
```

但大多数场景下，**可读性和安全性比微小性能更重要**，推荐用 `defer`。

---

## 八、如何检测数据竞争

Go 内置了 race detector：

```bash
go run -race main.go
go test -race ./...
go build -race
```

只要有数据竞争，运行时会打印：

```
==================
WARNING: DATA RACE
Write at 0x00c000014178 by goroutine 7:
  main.main.func1()
      main.go:15 +0x3c
...
==================
```

**开发阶段建议始终带 `-race`**，发现问题远比线上诡异 bug 好排查。

---

## 九、什么时候不需要 `Mutex`？

不是所有并发场景都要加锁，以下情况可以用更轻量的方案：

| 场景 | 方案 |
| :--- | :--- |
| 单个整数/指针的原子读写 | `sync/atomic` |
| 只需要执行一次 | `sync.Once` |
| 等待一组 goroutine 完成 | `sync.WaitGroup` |
| 只读共享数据（初始化后不再改） | 无需加锁 |
| goroutine 间通信 | `channel` |
| 简单的 map 并发访问 | `sync.Map`（特定场景） |

**Go 的哲学**：不要通过共享内存来通信，而要通过通信来共享内存。

如果能用 `channel` 优雅解决，就优先用 `channel`；`Mutex` 适合**保护一块明确的小临界区**。

---

## 十、一个完整可运行的例子

综合演示：多 goroutine 并发写入一个 `map`，用 `Mutex` 保护。

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	mu sync.Mutex
	m  map[string]int
}

func NewCounter() *Counter {
	return &Counter{m: make(map[string]int)}
}

func (c *Counter) Add(key string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.m[key]++
}

func (c *Counter) Snapshot() map[string]int {
	c.mu.Lock()
	defer c.mu.Unlock()
	snapshot := make(map[string]int, len(c.m))
	for k, v := range c.m {
		snapshot[k] = v
	}
	return snapshot
}

func main() {
	c := NewCounter()
	var wg sync.WaitGroup

	words := []string{"go", "rust", "go", "python", "go", "rust"}

	for _, w := range words {
		for i := 0; i < 1000; i++ {
			wg.Add(1)
			go func(word string) {
				defer wg.Done()
				c.Add(word)
			}(w)
		}
	}

	wg.Wait()

	for k, v := range c.Snapshot() {
		fmt.Printf("%-8s %d\n", k, v)
	}
}
```

输出：

```
go       3000
rust     2000
python   1000
```

用 `go run -race main.go` 运行也不会有任何警告。

---

## 十一、一句话总结

> **`sync.Mutex` 是保护共享数据的最小工具：`Lock` 进入临界区，`defer Unlock` 保证退出；锁和数据放在同一个结构体里，用指针接收者操作；临界区要尽量小；开发时始终带 `-race`。**

在你的 Prompt 工程里，日志写入、token 统计累加、多会话管理这些都是典型的加锁场景。目前的 `logger.Write` 已经用对了模式，可以直接参考。