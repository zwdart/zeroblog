---
title: Goroutine详细：并发编程详细讲解
date: 2026-10-10T10:02:47+08:00
lastmod: 2026-10-10T10:02:47+08:00
# avatar: /img/author.jpg
# authorlink: https://author.site
cover:
    image: "images/cover.svg"   # ✅ 正确：cover 是对象
    alt: "这里有一直迷路的小兔子"                # 可选：图片无法显示时的替代文本
    caption: "山高月小"                # 可选：图片下方的说明文字
    relative: false                 # 可选：设置为 true 时，路径相对于当前页面
# covercaption: a description of the cover image
# images:
#   - /img/cover.jpg
categories:
  - golang
tags:
  - Goroutine
# nolastmod: true
# math: true
draft: false
---

# Go 并发编程详细讲解

Go 的并发模型基于 **CSP（Communicating Sequential Processes）**，核心思想是：**不要通过共享内存来通信，而要通过通信来共享内存。**

下面从基础到进阶，系统讲解 Go 并发的所有核心概念，每个概念都配上可运行的示例。

---

## 一、Goroutine：轻量级线程

### 1.1 基础用法

```go
package main

import (
	"fmt"
	"time"
)

func say(s string) {
	for i := 0; i < 3; i++ {
		fmt.Println(s, i)
		time.Sleep(100 * time.Millisecond)
	}
}

func main() {
	go say("world")   // 启动一个 goroutine
	say("hello")      // 主 goroutine 继续执行

	time.Sleep(500 * time.Millisecond) // 等 goroutine 跑完
}
```

**关键点**：
- `go` 关键字启动一个 goroutine，**立即返回**，不阻塞。
- 主 goroutine 结束时，**所有其他 goroutine 都会被强制终止**。所以 `main` 里需要 `Sleep` 或同步机制等待。
- Goroutine 初始栈只有 **2KB**，可动态增长，所以可以轻松启动数十万个。

### 1.2 为什么 goroutine 比线程轻

| 对比项 | 线程 | Goroutine |
|--------|------|-----------|
| 初始栈大小 | 1-8 MB | 2 KB |
| 切换成本 | 内核态切换，微秒级 | 用户态切换，纳秒级 |
| 数量级 | 几千 | 百万级 |
| 调度 | 操作系统 | Go runtime（GMP 模型） |

### 1.3 用 WaitGroup 等待

`time.Sleep` 不靠谱，生产代码用 `sync.WaitGroup`：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 5; i++ {
		wg.Add(1)          // 计数 +1
		go func(n int) {
			defer wg.Done() // 计数 -1（defer 保证一定执行）
			fmt.Println("worker", n)
		}(i)               // 注意：i 作为参数传入，避免闭包捕获问题
	}

	wg.Wait()              // 阻塞直到计数归零
	fmt.Println("全部完成")
}
```

**易错点**：不要写成 `go func() { ... }(i)` 里的 `i` 直接捕获。Go 1.22 之前循环变量是共享的，需要用参数传入或 `i := i` 复制。Go 1.22 起每次迭代都是新变量，但显式传参更清晰。

---

## 二、Channel：goroutine 之间的通信管道

Channel 是 Go 并发的核心。可以把它想象成一条**类型安全的队列**。

### 2.1 无缓冲 Channel

```go
ch := make(chan int)   // 无缓冲

go func() {
	ch <- 42           // 发送：阻塞直到有人接收
}()
v := <-ch              // 接收：阻塞直到有人发送
fmt.Println(v)         // 42
```

**特点**：发送和接收**必须同时就绪**，否则阻塞。这叫**同步通信**。

### 2.2 有缓冲 Channel

```go
ch := make(chan int, 3)  // 缓冲大小 3

ch <- 1                  // 不阻塞
ch <- 2                  // 不阻塞
ch <- 3                  // 不阻塞
// ch <- 4               // 会阻塞，缓冲满了

fmt.Println(<-ch)        // 1
fmt.Println(<-ch)        // 2
fmt.Println(<-ch)        // 3
```

**特点**：缓冲未满时发送不阻塞，缓冲非空时接收不阻塞。缓冲满了发送阻塞，缓冲空了接收阻塞。

### 2.3 关闭 Channel

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
close(ch)

// 方式一：range 自动结束
for v := range ch {
	fmt.Println(v)   // 1, 2，然后循环自动结束
}

// 方式二：逗号 ok 判断
v, ok := <-ch
if !ok {
	fmt.Println("channel 已关闭")
}
```

**规则**：
- 只有**发送方**应该关闭 channel，接收方关闭会 panic。
- 关闭后**不能发送**，但**可以继续接收剩余数据**。
- 重复关闭会 panic。
- 向 nil channel 发送/接收会永久阻塞。

### 2.4 Channel 作为同步信号

```go
done := make(chan struct{})   // struct{} 不占内存

go func() {
	// 做一些工作
	fmt.Println("工作完成")
	close(done)                // 或 done <- struct{}{}
}()

<-done                         // 等待完成
```

`struct{}` 是零大小类型，作为纯信号使用时比 `bool` 更省内存。

### 2.5 Channel 方向

```go
func producer(out chan<- int) {   // 只发送
	for i := 0; i < 5; i++ {
		out <- i
	}
	close(out)
}

func consumer(in <-chan int) {    // 只接收
	for v := range in {
		fmt.Println(v)
	}
}

func main() {
	ch := make(chan int)
	go producer(ch)
	consumer(ch)
}
```

方向约束让编译器帮你检查错误，是良好的 API 设计习惯。

---

## 三、Select：多路复用

`select` 让一个 goroutine 同时等待多个 channel 操作。

### 3.1 基础用法

```go
select {
case v := <-ch1:
	fmt.Println("收到 ch1:", v)
case ch2 <- 42:
	fmt.Println("发送到 ch2")
case <-time.After(time.Second):
	fmt.Println("超时")
default:
	fmt.Println("都没有就绪，立即执行")
}
```

**规则**：
- 多个 case 同时就绪时，**随机选一个**（防止饥饿）。
- 有 `default` 时**不阻塞**；没有 `default` 时**阻塞**直到某个 case 就绪。

### 3.2 超时控制

```go
func fetchData() (string, error) {
	ch := make(chan string, 1)

	go func() {
		time.Sleep(2 * time.Second)  // 模拟慢操作
		ch <- "data"
	}()

	select {
	case v := <-ch:
		return v, nil
	case <-time.After(1 * time.Second):
		return "", fmt.Errorf("超时")
	}
}
```

### 3.3 非阻塞接收

```go
select {
case v := <-ch:
	fmt.Println("收到:", v)
default:
	fmt.Println("没有数据")
}
```

### 3.4 退出信号

```go
func worker(stop <-chan struct{}) {
	for {
		select {
		case <-stop:
			fmt.Println("收到退出信号")
			return
		default:
			// 正常工作
			time.Sleep(100 * time.Millisecond)
		}
	}
}
```

---

## 四、Context：取消与超时

`context` 是 Go 官方推荐的**跨 goroutine 传递取消信号和超时**的机制。

### 4.1 四种 Context

```go
ctx := context.Background()                          // 根 context，永不取消
ctx, cancel := context.WithCancel(parent)            // 手动取消
ctx, cancel := context.WithTimeout(parent, 3*time.Second)  // 超时取消
ctx, cancel := context.WithDeadline(parent, time.Now().Add(3*time.Second)) // 截止时间
defer cancel()                                        // 必须调用，防止资源泄漏
```

### 4.2 取消传播

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())

	go worker(ctx, "A")
	go worker(ctx, "B")

	time.Sleep(2 * time.Second)
	cancel()   // 取消信号传播到所有子 goroutine

	time.Sleep(500 * time.Millisecond)
}

func worker(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name, "退出:", ctx.Err())
			return
		default:
			fmt.Println(name, "工作中")
			time.Sleep(500 * time.Millisecond)
		}
	}
}
```

### 4.3 配合 HTTP 请求

```go
func callAPI(ctx context.Context) error {
	req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com", nil)
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err  // 如果 ctx 被取消，err 会是 context.Canceled
	}
	defer resp.Body.Close()
	// ...
	return nil
}
```

### 4.4 Context 使用原则

| 原则 | 说明 |
|------|------|
| 作为函数**第一个参数** | `func Do(ctx context.Context, ...)` |
| 不放进 struct | Context 应该随调用链传递，不是对象状态 |
| 只用于**取消、超时、传值** | 不要用它传业务参数 |
| `cancel()` 必须调用 | `defer cancel()` 防止 goroutine 泄漏 |
| `context.Value` 少用 | 只用于 request-scoped 的元数据（如 traceID、用户身份） |

---

## 五、sync 包：共享内存同步

虽然 Go 提倡用 channel 通信，但有些场景用共享内存更自然。

### 5.1 Mutex：互斥锁

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}

func (c *Counter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.n
}
```

### 5.2 RWMutex：读写锁

读多写少的场景，允许多个读者并发：

```go
type Cache struct {
	mu sync.RWMutex
	m  map[string]string
}

func (c *Cache) Get(k string) (string, bool) {
	c.mu.RLock()          // 读锁，可多个 goroutine 同时持有
	defer c.mu.RUnlock()
	v, ok := c.m[k]
	return v, ok
}

func (c *Cache) Set(k, v string) {
	c.mu.Lock()           // 写锁，独占
	defer c.mu.Unlock()
	c.m[k] = v
}
```

### 5.3 Once：只执行一次

```go
var (
	instance *DB
	once     sync.Once
)

func GetDB() *DB {
	once.Do(func() {
		instance = &DB{}   // 只执行一次，其他 goroutine 阻塞等待
	})
	return instance
}
```

单例模式的标准实现，比手动加锁更安全。

### 5.4 sync.Map：并发安全的 Map

普通 map 并发读写会 panic。`sync.Map` 适合**读多写少、key 相对固定**的场景：

```go
var m sync.Map

m.Store("key", "value")
v, ok := m.Load("key")
m.Delete("key")
m.Range(func(k, v any) bool {
	fmt.Println(k, v)
	return true
})
```

**注意**：`sync.Map` 不是万能替代。大部分场景用 `sync.RWMutex + map` 性能更好。`sync.Map` 只在特定场景（如缓存、注册表）有优势。

### 5.5 sync.Pool：对象复用

减少 GC 压力，适合频繁创建销毁的对象：

```go
var bufPool = sync.Pool{
	New: func() any {
		return new(bytes.Buffer)
	},
}

func process() {
	buf := bufPool.Get().(*bytes.Buffer)
	defer func() {
		buf.Reset()
		bufPool.Put(buf)
	}()
	buf.WriteString("data")
	// ...
}
```

**注意**：`sync.Pool` 里的对象可能在任意 GC 时被回收，不能用于持久化状态。

---

## 六、经典并发模式

### 6.1 Worker Pool（工作池）

控制并发数量，避免资源耗尽：

```go
func workerPool() {
	jobs := make(chan int, 100)
	results := make(chan int, 100)

	// 启动 3 个 worker
	var wg sync.WaitGroup
	for w := 0; w < 3; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range jobs {
				results <- j * j   // 处理任务
			}
		}()
	}

	// 发送任务
	go func() {
		for i := 0; i < 10; i++ {
			jobs <- i
		}
		close(jobs)
	}()

	// 等待 worker 完成后关闭 results
	go func() {
		wg.Wait()
		close(results)
	}()

	// 收集结果
	for r := range results {
		fmt.Println(r)
	}
}
```

### 6.2 Fan-out / Fan-in（扇出扇入）

```go
// Fan-out：一个输入分发给多个 worker
// Fan-in：多个 worker 输出汇聚成一个 channel

func fanIn(chs ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	for _, ch := range chs {
		wg.Add(1)
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c {
				out <- v
			}
		}(ch)
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}
```

### 6.3 Pipeline（流水线）

```go
func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)
	}()
	return out
}

func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			out <- n * n
		}
		close(out)
	}()
	return out
}

func main() {
	// gen → square → 输出
	for v := range square(gen(1, 2, 3, 4, 5)) {
		fmt.Println(v)   // 1, 4, 9, 16, 25
	}
}
```

### 6.4 限流（Semaphore）

```go
func limited() {
	sem := make(chan struct{}, 3)   // 最多 3 个并发
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			sem <- struct{}{}        // 获取令牌
			defer func() { <-sem }() // 释放令牌

			fmt.Println("处理", n)
			time.Sleep(time.Second)
		}(i)
	}
	wg.Wait()
}
```

用 `golang.org/x/sync/semaphore` 可以支持带权重的限流。

### 6.5 errgroup：带错误传播的并发

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) error {
	g, ctx := errgroup.WithContext(ctx)

	for _, url := range urls {
		url := url
		g.Go(func() error {
			return fetch(ctx, url)   // 任一失败会取消 ctx
		})
	}
	return g.Wait()   // 返回第一个错误
}
```

`errgroup` 是并发任务编排的利器，比手写 WaitGroup + channel 简洁得多。

---

## 七、数据竞争与检测

### 7.1 竞争示例

```go
func main() {
	n := 0
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			n++   // 数据竞争！
		}()
	}
	wg.Wait()
	fmt.Println(n)   // 通常小于 1000
}
```

### 7.2 用 -race 检测

```bash
go run -race main.go
go test -race ./...
```

`-race` 会在运行时检测数据竞争，输出冲突的读写位置。**生产构建不要开 -race**（性能损失约 10 倍），但开发和 CI 一定要开。

### 7.3 修复方式

**方式一：加锁**
```go
var mu sync.Mutex
mu.Lock()
n++
mu.Unlock()
```

**方式二：用 atomic**
```go
var n int64
atomic.AddInt64(&n, 1)
```

**方式三：用 channel**
```go
ch := make(chan int)
go func() { ch <- 1 }()
n += <-ch
```

---

## 八、GMP 调度模型

理解 Go 调度的核心：

| 组件 | 含义 |
|------|------|
| **G** (Goroutine) | 用户态协程，包含栈、指令指针等 |
| **M** (Machine) | 操作系统线程 |
| **P** (Processor) | 逻辑处理器，持有可运行 G 的本地队列 |

**调度流程**：
1. 每个 P 有一个本地运行队列，M 绑定 P 后从中取 G 执行。
2. 本地队列空时，M 会从全局队列或其他 P 的队列**窃取** G（work stealing）。
3. G 遇到阻塞系统调用时，M 会释放 P，让其他 M 绑定 P 继续执行。
4. 网络 I/O 由 netpoller 处理，不阻塞 M。

**关键参数**：
- `GOMAXPROCS`：P 的数量，默认等于 CPU 核数。
- `runtime.NumGoroutine()`：当前 goroutine 数量，用于排查泄漏。

---

## 九、常见坑与最佳实践

### 9.1 Goroutine 泄漏

```go
// 错误：goroutine 永远阻塞
func bad() {
	ch := make(chan int)
	go func() {
		ch <- 42   // 没人接收，永久阻塞
	}()
	// 函数返回，goroutine 泄漏
}

// 正确：用 context 或 buffered channel
func good(ctx context.Context) {
	ch := make(chan int, 1)   // 缓冲 1，发送不阻塞
	go func() {
		select {
		case ch <- 42:
		case <-ctx.Done():
		}
	}()
}
```

### 9.2 循环变量捕获

```go
// Go 1.22 之前：所有 goroutine 打印 5
for i := 0; i < 5; i++ {
	go func() { fmt.Println(i) }()
}

// 修复方式一：传参
for i := 0; i < 5; i++ {
	go func(n int) { fmt.Println(n) }(i)
}

// 修复方式二：局部变量
for i := 0; i < 5; i++ {
	i := i
	go func() { fmt.Println(i) }()
}
```

Go 1.22 起循环变量每次迭代都是新的，不再有此问题。

### 9.3 关闭已关闭的 Channel

```go
// 错误：重复关闭 panic
close(ch)
close(ch)   // panic: close of closed channel
```

用 `sync.Once` 保证只关一次：

```go
var once sync.Once
once.Do(func() { close(ch) })
```

### 9.4 向已关闭的 Channel 发送

```go
close(ch)
ch <- 1   // panic: send on closed channel
```

**规则**：只有发送方关闭，且关闭后不再发送。

### 9.5 用 Buffered Channel 避免泄漏

如果 goroutine 发送后可能没人接收，用缓冲 channel：

```go
ch := make(chan int, 1)   // 缓冲 1，发送方不会阻塞
go func() { ch <- 42 }()
```

### 9.6 超时与重试

```go
func retry(ctx context.Context, fn func() error) error {
	for i := 0; i < 3; i++ {
		if err := fn(); err == nil {
			return nil
		}
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-time.After(time.Duration(i+1) * time.Second):
		}
	}
	return fmt.Errorf("重试 3 次仍失败")
}
```

---

## 十、决策指南：用 Channel 还是 Mutex

| 场景 | 推荐 |
|------|------|
| 传递数据所有权 | Channel |
| 分发任务给多个 worker | Channel |
| 通知事件（signal） | Channel |
| 保护共享状态（计数器、缓存） | Mutex / RWMutex |
| 单例初始化 | sync.Once |
| 对象池 | sync.Pool |
| 并发 map | sync.Map 或 RWMutex + map |

**简单判断**：如果你在问"谁拥有这份数据"，用 Channel；如果你在问"谁能同时访问这份数据"，用 Mutex。

---

## 十一、AI 应用中的并发模式

结合你之前的项目，几个实际场景：

### 11.1 流式响应 + 超时

```go
func streamWithTimeout(ctx context.Context, client *llm.Client) error {
	ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
	defer cancel()

	_, err := client.ChatStream(ctx, msgs, func(tok string) {
		fmt.Print(tok)
	})
	return err
}
```

### 11.2 并发调用多个模型对比

```go
func compareModels(ctx context.Context, prompt string, models []string) (map[string]string, error) {
	g, ctx := errgroup.WithContext(ctx)
	results := sync.Map{}

	for _, m := range models {
		m := m
		g.Go(func() error {
			client := llm.NewClient(baseURL, apiKey, m, httpClient)
			reply, err := client.Chat(ctx, []llm.Message{{Role: "user", Content: prompt}})
			if err != nil {
				return err
			}
			results.Store(m, reply)
			return nil
		})
	}

	if err := g.Wait(); err != nil {
		return nil, err
	}

	out := map[string]string{}
	results.Range(func(k, v any) bool {
		out[k.(string)] = v.(string)
		return true
	})
	return out, nil
}
```

### 11.3 Token 计数限流

```go
type TokenLimiter struct {
	sem chan struct{}
}

func NewTokenLimiter(maxConcurrent int) *TokenLimiter {
	return &TokenLimiter{sem: make(chan struct{}, maxConcurrent)}
}

func (l *TokenLimiter) Acquire(ctx context.Context) error {
	select {
	case l.sem <- struct{}{}:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (l *TokenLimiter) Release() {
	<-l.sem
}
```

---

## 十二、总结

| 主题 | 核心要点 |
|------|----------|
| **Goroutine** | 轻量、go 关键字、主 goroutine 退出会杀死所有 |
| **Channel** | 无缓冲同步、有缓冲异步、关闭规则、方向约束 |
| **Select** | 多路复用、超时、default 非阻塞 |
| **Context** | 取消传播、超时、作为第一个参数 |
| **sync 包** | Mutex/RWMutex/Once/Pool/Map |
| **经典模式** | Worker Pool、Fan-in/out、Pipeline、Semaphore、errgroup |
| **数据竞争** | 用 -race 检测，加锁或 atomic 修复 |
| **GMP** | G=协程、M=线程、P=逻辑处理器，work stealing |
| **常见坑** | goroutine 泄漏、循环变量、重复关闭、死锁 |
| **决策** | 传数据用 channel，护状态用 mutex |

**一句话**：Go 并发的精髓是 **"用 channel 传递数据，用 context 传递取消，用 sync 保护状态"**。理解这三者的适用场景，就能写出健壮的并发程序。