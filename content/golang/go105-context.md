---
title: 'Goroutine Context 详细讲解'
date: '2026-10-12T11:21:07+08:00'
categories: ['golang']
tags: ['goroutine context']
description: 'desc'
draft: false

cover:
  image: 'images/cover.svg'
  alt: '山高月小，水落石出。'
  caption: '山水'
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
---

# Go Context 详细讲解

`context` 是 Go 官方用来**在 goroutine 之间传递取消信号、超时、截止时间和请求范围数据**的标准库。它解决了并发编程中的核心问题：**如何优雅地通知一组 goroutine 停止工作**。

---

## 一、为什么需要 Context

### 1.1 没有 Context 的痛点

```go
// 假设一个 HTTP 请求启动了 3 个 goroutine 处理子任务
func handler(w http.ResponseWriter, r *http.Request) {
    go fetchFromDB()      // 用户断开连接，它还在跑
    go callExternalAPI()  // 超时了，它还在跑
    go writeToCache()     // 没人需要结果了，它还在跑
}
```

问题：
- 用户取消请求，goroutine 收不到通知，继续浪费资源
- 无法设置统一超时
- 无法在调用链中传递请求 ID、用户身份等数据
- goroutine 泄漏

### 1.2 Context 的设计哲学

> **"一个请求，一棵 goroutine 树，一个 Context。"**

- Context 是**树形结构**，根节点是 `context.Background()` 或 `context.TODO()`
- 父 Context 取消 → **所有子 Context 全部取消**（级联）
- Context 是**并发安全**的，可被多个 goroutine 同时使用
- Context 应该作为**函数的第一个参数**，命名为 `ctx`

---

## 二、Context 接口

```go
type Context interface {
    // 返回 deadline 和是否设置了 deadline
    Deadline() (deadline time.Time, ok bool)

    // 返回一个 channel，在 Context 被取消时关闭
    Done() <-chan struct{}

    // 返回取消原因（在 Done 关闭后才有意义）
    Err() error

    // 获取 key 对应的值
    Value(key any) any
}
```

**关键点：**
- `Done()` 返回的 channel 在取消/超时时**关闭**
- `Err()` 返回 `context.Canceled` 或 `context.DeadlineExceeded`
- 没有取消时，`Done()` 返回 `nil`（会永久阻塞）

---

## 三、四种创建方式

### 3.1 根 Context

```go
ctx := context.Background() // 通常用于 main、初始化、测试
ctx := context.TODO()       // 不确定用哪个时，占位
```

两者都是空的、永不取消的 Context。**生产代码中只用 Background，TODO 只是标记待办。**

### 3.2 WithCancel：手动取消

```go
ctx, cancel := context.WithCancel(parentCtx)
defer cancel() // 必须调用，否则泄漏
```

### 3.3 WithTimeout：超时取消

```go
ctx, cancel := context.WithTimeout(parentCtx, 2*time.Second)
defer cancel()
```

### 3.4 WithDeadline：到某时刻取消

```go
ctx, cancel := context.WithDeadline(parentCtx, time.Now().Add(2*time.Second))
defer cancel()
```

### 3.5 WithValue：携带数据

```go
ctx = context.WithValue(parentCtx, key, value)
```

---

## 四、基础示例

### 4.1 WithCancel 基本用法

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("worker %d 收到取消信号: %v\n", id, ctx.Err())
            return
        default:
            fmt.Printf("worker %d 工作中...\n", id)
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    go worker(ctx, 1)
    go worker(ctx, 2)

    time.Sleep(2 * time.Second)
    fmt.Println("main: 发送取消信号")
    cancel() // 通知所有子 goroutine

    time.Sleep(500 * time.Millisecond)
    fmt.Println("main: 退出")
}
```

**输出：**
```
worker 1 工作中...
worker 2 工作中...
worker 1 工作中...
worker 2 工作中...
main: 发送取消信号
worker 1 收到取消信号: context canceled
worker 2 收到取消信号: context canceled
main: 退出
```

---

### 4.2 WithTimeout 超时控制

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func slowOperation(ctx context.Context) (string, error) {
    select {
    case <-time.After(3 * time.Second):
        return "操作完成", nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    result, err := slowOperation(ctx)
    if err != nil {
        fmt.Println("失败:", err) // 失败: context deadline exceeded
        return
    }
    fmt.Println("成功:", result)
}
```

---

### 4.3 级联取消（树形结构）

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    root, cancelRoot := context.WithCancel(context.Background())
    defer cancelRoot()

    child1, cancelChild1 := context.WithCancel(root)
    defer cancelChild1()

    child2, _ := context.WithCancel(root)

    go func() {
        <-child1.Done()
        fmt.Println("child1 被取消:", child1.Err())
    }()

    go func() {
        <-child2.Done()
        fmt.Println("child2 被取消:", child2.Err())
    }()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("取消 root")
    cancelRoot() // 级联取消 child1 和 child2

    time.Sleep(200 * time.Millisecond)
}
```

**输出：**
```
取消 root
child1 被取消: context canceled
child2 被取消: context canceled
```

> **规则**：取消父节点 → 所有子节点自动取消；取消子节点 → 不影响父节点。

---

## 五、WithValue 传值

### 5.1 基本用法

```go
package main

import (
    "context"
    "fmt"
)

// key 必须是不导出的自定义类型，避免冲突
type contextKey string

const (
    userIDKey    contextKey = "userID"
    requestIDKey contextKey = "requestID"
)

func handleRequest(ctx context.Context) {
    if userID, ok := ctx.Value(userIDKey).(string); ok {
        fmt.Println("用户:", userID)
    }
    if reqID, ok := ctx.Value(requestIDKey).(string); ok {
        fmt.Println("请求 ID:", reqID)
    }
}

func main() {
    ctx := context.Background()
    ctx = context.WithValue(ctx, userIDKey, "user-123")
    ctx = context.WithValue(ctx, requestIDKey, "req-456")

    handleRequest(ctx)
}
```

**输出：**
```
用户: user-123
请求 ID: req-456
```

### 5.2 为什么 key 不能用 string？

```go
// ❌ 错误：不同包可能用相同 string 键冲突
ctx = context.WithValue(ctx, "userID", "123")

// ✅ 正确：用自定义类型
type ctxKey int
const userIDKey ctxKey = 0
ctx = context.WithValue(ctx, userIDKey, "123")
```

### 5.3 WithValue 的适用场景

**✅ 适合：**
- 请求 ID、trace ID（链路追踪）
- 用户身份、租户 ID
- 认证 token
- 跨 API 边界的请求范围元数据

**❌ 不适合：**
- 传递函数参数（应该显式传参）
- 传递可选参数
- 传递可变状态

> 官方原话：*"Use context values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions."*

---

## 六、在 goroutine 中正确使用

### 6.1 模式一：worker 监听 ctx

```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case <-ctx.Done():
            return
        case job, ok := <-jobs:
            if !ok {
                return
            }
            process(job)
        }
    }
}
```

### 6.2 模式二：启动多个 goroutine 并统一取消

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    fmt.Printf("worker %d 退出\n", id)
                    return
                default:
                    time.Sleep(300 * time.Millisecond)
                }
            }
        }(i)
    }

    time.Sleep(1 * time.Second)
    cancel()
    wg.Wait()
    fmt.Println("所有 worker 已退出")
}
```

### 6.3 模式三：ErrGroup（推荐）

`golang.org/x/sync/errgroup` 结合了 WaitGroup 和 Context，任何 goroutine 出错会取消其他 goroutine：

```go
package main

import (
    "context"
    "fmt"
    "time"

    "golang.org/x/sync/errgroup"
)

func main() {
    g, ctx := errgroup.WithContext(context.Background())

    g.Go(func() error {
        select {
        case <-time.After(3 * time.Second):
            return nil
        case <-ctx.Done():
            return ctx.Err()
        }
    })

    g.Go(func() error {
        time.Sleep(1 * time.Second)
        return fmt.Errorf("出错了") // 会取消其他 goroutine
    })

    if err := g.Wait(); err != nil {
        fmt.Println("errgroup 结束:", err)
    }
}
```

---

## 七、实战场景

### 7.1 HTTP 请求超时

```go
func fetchURL(url string) (string, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return "", err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    return string(body), nil
}
```

### 7.2 HTTP Server 中传递 Context

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // HTTP 请求自带的 Context

    // 用户断开连接时 ctx 会自动取消
    result, err := doSomething(ctx)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // 客户端断开
            return
        }
        http.Error(w, err.Error(), 500)
        return
    }
    w.Write([]byte(result))
}

func doSomething(ctx context.Context) (string, error) {
    select {
    case <-ctx.Done():
        return "", ctx.Err()
    case <-time.After(5 * time.Second):
        return "done", nil
    }
}
```

### 7.3 数据库查询超时

```go
func queryDB(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    rows, err := db.QueryContext(ctx, "SELECT * FROM users")
    if err != nil {
        return err
    }
    defer rows.Close()
    // ...
    return nil
}
```

### 7.4 链式调用取消

```go
func serviceA(ctx context.Context) error {
    return serviceB(ctx) // 传递 ctx
}

func serviceB(ctx context.Context) error {
    return serviceC(ctx)
}

func serviceC(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err() // 任意一层取消，全部返回
    case <-time.After(1 * time.Second):
        return nil
    }
}
```

### 7.5 后台任务 + 优雅关闭

```go
package main

import (
    "context"
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 监听系统信号
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        <-sigCh
        fmt.Println("\n收到退出信号")
        cancel()
    }()

    fmt.Println("服务已启动，Ctrl+C 退出")
    <-ctx.Done()
    fmt.Println("正在清理...")
    time.Sleep(500 * time.Millisecond)
    fmt.Println("已退出")
}
```

---

## 八、常见陷阱

### 8.1 忘记调用 cancel（goroutine 泄漏）

```go
// ❌ 即使操作提前结束，Context 也不会释放
func bad() {
    ctx, _ := context.WithTimeout(context.Background(), time.Hour)
    doSomething(ctx)
}

// ✅
func good() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
    defer cancel()
    doSomething(ctx)
}
```

> **规则**：永远 `defer cancel()`，即使不超时也要调用。

### 8.2 把 Context 存到 struct 里

```go
// ❌ Context 应该作为函数参数传递
type Service struct {
    ctx context.Context
}

// ✅
type Service struct{}
func (s *Service) Do(ctx context.Context) error { ... }
```

### 8.3 用 Context 传普通参数

```go
// ❌
ctx = context.WithValue(ctx, "userID", 123)

// ✅ 显式传参
func getUser(ctx context.Context, userID int) (*User, error)
```

### 8.4 在 select 中忽略 ctx

```go
// ❌ 无法取消
for {
    select {
    case job := <-jobs:
        process(job)
    }
}

// ✅
for {
    select {
    case <-ctx.Done():
        return
    case job := <-jobs:
        process(job)
    }
}
```

### 8.5 检查 ctx.Err() 的时机

```go
select {
case <-ctx.Done():
    // 此时 ctx.Err() 才有值
    fmt.Println(ctx.Err()) // context.Canceled 或 context.DeadlineExceeded
}
```

### 8.6 取消不会杀死 goroutine

Context 只是**信号**，需要 goroutine 主动检查 `ctx.Done()` 并退出：

```go
go func() {
    for {
        select {
        case <-ctx.Done():
            return // 必须主动返回
        default:
            // 注意：如果这里有阻塞操作，仍需该操作支持 ctx
        }
    }
}()
```

---

## 九、Context 底层结构

```go
type emptyCtx int // Background / TODO

type cancelCtx struct {
    Context
    mu       sync.Mutex
    done     atomic.Value // chan struct{}
    children map[canceler]struct{}
    err      error
}

type timerCtx struct {
    cancelCtx
    timer    *time.Timer
    deadline time.Time
}

type valueCtx struct {
    Context
    key, val any
}
```

**关键机制：**
- `cancelCtx` 维护**子节点列表**，取消时遍历通知所有子节点
- `timerCtx` 内嵌 `cancelCtx`，额外带 `timer` 和 `deadline`
- `valueCtx` 是**链表**结构，`Value()` 沿链向上查找
- `Done()` 返回的 channel **懒加载**，只有在第一次调用时才创建

---

## 十、Context 使用清单

| 规则 | 说明 |
|------|------|
| 第一个参数 | `func Do(ctx context.Context, ...)` |
| 不要存 struct | Context 作为参数传递 |
| 命名 ctx | 参数统一叫 `ctx` |
| 必 defer cancel | 即使提前返回也要释放 |
| nil 用 TODO | 不确定时用 `context.TODO()` |
| 不放 nil | 不要传 nil Context |
| key 用自定义类型 | 避免包间冲突 |
| 只放请求范围数据 | 不放参数、可选值 |
| Done 用 select | 配合 select 监听取消 |

---

## 十一、Context 的替代与补充

| 场景 | 推荐 |
|------|------|
| 单个 goroutine 取消 | `context.WithCancel` |
| 多个 goroutine 统一取消 | `errgroup` + context |
| 纯等待完成 | `sync.WaitGroup`（不需要 ctx） |
| 只有超时 | `time.After`（不需要 ctx） |
| 需要传值 + 取消 | `context.WithValue` + `WithTimeout` |

---

## 十二、完整实战：带超时的并发请求

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"

    "golang.org/x/sync/errgroup"
)

type Result struct {
    Source string
    Data   string
}

func fetch(ctx context.Context, source string, delay time.Duration) (*Result, error) {
    select {
    case <-time.After(delay):
        return &Result{Source: source, Data: source + "-data"}, nil
    case <-ctx.Done():
        return nil, fmt.Errorf("%s: %w", source, ctx.Err())
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1500*time.Millisecond)
    defer cancel()

    g, ctx := errgroup.WithContext(ctx)

    sources := map[string]time.Duration{
        "db":      300 * time.Millisecond,
        "cache":   200 * time.Millisecond,
        "api":     2 * time.Second, // 会超时
    }

    var mu sync.Mutex
    results := make(map[string]string)

    for name, delay := range sources {
        name, delay := name, delay
        g.Go(func() error {
            r, err := fetch(ctx, name, delay)
            if err != nil {
                return err
            }
            mu.Lock()
            results[r.Source] = r.Data
            mu.Unlock()
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        fmt.Println("有任务失败:", err)
    }

    fmt.Println("已完成的任务:")
    for k, v := range results {
        fmt.Printf("  %s: %s\n", k, v)
    }
}
```

**输出示例：**
```
有任务失败: api: context deadline exceeded
已完成的任务:
  cache: cache-data
  db: db-data
```

---

## 十三、核心要点总结

1. **Context 是取消信号传播树**，父取消 → 子全取消
2. **取消 ≠ 杀死 goroutine**，必须 goroutine 主动检查 `Done()`
3. **永远 `defer cancel()`**，防止泄漏
4. **Context 作为第一个参数**，不存 struct
5. **WithValue 只放请求范围数据**（trace ID、用户身份），不传参数
6. **HTTP / DB / RPC 都原生支持 Context**，直接传下去
7. **errgroup** 是 context + goroutine 的最佳实践组合
8. **WithValue 的 key 用自定义类型**，避免冲突

---

进一步展开：
- **`errgroup` 源码与高级用法**（SetLimit、TryGo）
- **`context` 源码分析**（`cancelCtx` 的传播机制）
- **分布式链路追踪**（OpenTelemetry + context）
- **gRPC 中的 Context 传递**