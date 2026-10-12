---
title: 'Goroutine Select'
date: '2026-10-12T09:55:20+08:00'
categories: ['golang']
tags: ['select']
description: 'Go select 详细讲解，是 Go 专门为 channel 设计的多路复用语句，类似 `switch`，但每个 case 都是 channel 操作。它让一个 goroutine 能同时等待多个 channel 的读写。'
draft: true

cover:
  image: 'images/cover.svg'
  alt: '山高月小，水落石出。'
  caption: '山水'
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
---



# Go select 详细讲解

`select` 是 Go 专门为 channel 设计的多路复用语句，类似 `switch`，但每个 case 都是 channel 操作。它让一个 goroutine 能同时等待多个 channel 的读写。

---

## 一、基本语法

```go
select {
case v := <-ch1:
    // ch1 有数据可读
case ch2 <- x:
    // ch2 可写入
case v, ok := <-ch3:
    // 带关闭检测
default:
    // 所有 case 都未就绪时执行
}
```

**核心规则：**
1. 每个 case 必须是一个 channel 操作（发送或接收）
2. 多个 case 同时就绪时，**随机选一个**（保证公平）
3. 没有 `default` 时，`select` 会**阻塞**直到某个 case 就绪
4. 有 `default` 时，立即执行，不阻塞

---

## 二、最基础示例

### 2.1 同时等待两个 channel

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "来自 ch1"
    }()

    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "来自 ch2"
    }()

    for i := 0; i < 2; i++ {
        select {
        case msg := <-ch1:
            fmt.Println(msg)
        case msg := <-ch2:
            fmt.Println(msg)
        }
    }
}
```

**输出：**
```
来自 ch1
来自 ch2
```

> 谁先就绪谁先执行，两个都就绪则随机。

---

## 三、default：非阻塞 select

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 1)

    // 有数据
    ch <- 42
    select {
    case v := <-ch:
        fmt.Println("读到:", v)
    default:
        fmt.Println("没数据")
    }

    // 无数据
    select {
    case v := <-ch:
        fmt.Println("读到:", v)
    default:
        fmt.Println("没数据")
    }
}
```

**输出：**
```
读到: 42
没数据
```

### 3.1 非阻塞发送

```go
select {
case ch <- 100:
    fmt.Println("发送成功")
default:
    fmt.Println("channel 满了，跳过")
}
```

### 3.2 轮询模式

```go
for {
    select {
    case v := <-ch:
        handle(v)
    default:
        // 做其他事情，不阻塞
        doSomethingElse()
        time.Sleep(10 * time.Millisecond)
    }
}
```

> ⚠️ 轮询会消耗 CPU，通常用 `time.Ticker` 或阻塞 select 替代。

---

## 四、超时控制（最常用）

### 4.1 `time.After`

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)

    select {
    case v := <-ch:
        fmt.Println("收到:", v)
    case <-time.After(2 * time.Second):
        fmt.Println("超时，2 秒内没收到数据")
    }
}
```

**输出：**
```
超时，2 秒内没收到数据
```

### 4.2 请求超时模式

```go
func doRequest(url string) (string, error) {
    result := make(chan string, 1)

    go func() {
        // 模拟耗时请求
        time.Sleep(3 * time.Second)
        result <- "响应数据"
    }()

    select {
    case r := <-result:
        return r, nil
    case <-time.After(1 * time.Second):
        return "", fmt.Errorf("请求超时")
    }
}
```

> ⚠️ `time.After` 每次都会创建定时器，在**循环**中会产生大量垃圾。循环内应改用 `time.NewTimer` + `Reset`。

### 4.3 循环中使用 Timer（避免泄漏）

```go
timer := time.NewTimer(1 * time.Second)
defer timer.Stop()

for {
    select {
    case v := <-ch:
        fmt.Println("收到:", v)
        // 重置
        if !timer.Stop() {
            <-timer.C
        }
        timer.Reset(1 * time.Second)
    case <-timer.C:
        fmt.Println("空闲超时")
        return
    }
}
```

---

## 五、for + select 经典模式

`select` 通常放在 `for` 里持续监听。

### 5.1 持续消费直到 channel 关闭

```go
package main

import "fmt"

func producer(ch chan int) {
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch)
}

func main() {
    ch := make(chan int)
    go producer(ch)

    for {
        v, ok := <-ch
        if !ok {
            fmt.Println("channel 已关闭")
            return
        }
        fmt.Println("收到:", v)
    }
}
```

### 5.2 select + 关闭检测

```go
for {
    select {
    case v, ok := <-ch:
        if !ok {
            fmt.Println("关闭了")
            return
        }
        fmt.Println(v)
    }
}
```

---

## 六、处理多个 channel（合并 / Fan-in）

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go func() {
        for i := 1; i <= 3; i++ {
            ch1 <- i
            time.Sleep(300 * time.Millisecond)
        }
        close(ch1)
    }()

    go func() {
        for i := 10; i <= 12; i++ {
            ch2 <- i
            time.Sleep(200 * time.Millisecond)
        }
        close(ch2)
    }()

    // 合并两个 channel
    for ch1 != nil || ch2 != nil {
        select {
        case v, ok := <-ch1:
            if !ok {
                ch1 = nil // 置 nil 后该 case 永久阻塞
                continue
            }
            fmt.Println("ch1:", v)
        case v, ok := <-ch2:
            if !ok {
                ch2 = nil
                continue
            }
            fmt.Println("ch2:", v)
        }
    }
    fmt.Println("全部完成")
}
```

**关键技巧：`ch = nil` 禁用某个 case**

- 从 nil channel 接收会永久阻塞
- 因此把已关闭的 channel 置为 nil，`select` 就不会再选中它
- 当所有 channel 都置 nil，且没有 default，会 panic：`all goroutines are asleep - deadlock!`

---

## 七、退出信号 / 优雅关闭

### 7.1 done channel

```go
package main

import (
    "fmt"
    "time"
)

func worker(done <-chan struct{}, data <-chan int) {
    for {
        select {
        case <-done:
            fmt.Println("收到退出信号，worker 停止")
            return
        case v := <-data:
            fmt.Println("处理:", v)
        }
    }
}

func main() {
    done := make(chan struct{})
    data := make(chan int)

    go worker(done, data)

    go func() {
        for i := 1; i <= 5; i++ {
            data <- i
            time.Sleep(200 * time.Millisecond)
        }
    }()

    time.Sleep(1 * time.Second)
    close(done) // 广播退出信号
    time.Sleep(100 * time.Millisecond)
}
```

> 用 `close(done)` 而不是 `done <- struct{}{}`，可以**广播**给所有监听的 goroutine。

### 7.2 context 替代方案（推荐）

```go
func worker(ctx context.Context, data <-chan int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("ctx 取消:", ctx.Err())
            return
        case v := <-data:
            fmt.Println("处理:", v)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel()

    data := make(chan int)
    go worker(ctx, data)
    // ...
}
```

---

## 八、select 的随机性

多个 case 同时就绪时，`select` **伪随机**选择，避免饥饿：

```go
package main

import "fmt"

func main() {
    ch1 := make(chan int, 1)
    ch2 := make(chan int, 1)

    ch1 <- 1
    ch2 <- 2

    for i := 0; i < 10; i++ {
        select {
        case v := <-ch1:
            fmt.Println("ch1:", v)
            ch1 <- 1
        case v := <-ch2:
            fmt.Println("ch2:", v)
            ch2 <- 2
        }
    }
}
```

**输出示例（顺序随机）：**
```
ch2: 2
ch1: 1
ch2: 2
ch1: 1
...
```

---

## 九、空 select

```go
select {}
```

**永久阻塞**，常用于 `main` 中等待其他 goroutine（不推荐，容易死锁误判）：

```go
func main() {
    go func() {
        for {
            // 后台任务
        }
    }()
    select {} // 主 goroutine 永久阻塞
}
```

> ⚠️ 运行时会报 `all goroutines are asleep - deadlock!` 只有在所有 goroutine 都阻塞时才报；如果后台 goroutine 在跑则不会。

---

## 十、优先级 select

`select` 是随机的，如果想给某个 channel 更高优先级，用**嵌套 select**：

```go
for {
    // 优先处理高优先级 channel
    select {
    case v := <-highPriority:
        handleHigh(v)
        continue
    default:
    }

    // 否则处理任意一个
    select {
    case v := <-highPriority:
        handleHigh(v)
    case v := <-lowPriority:
        handleLow(v)
    }
}
```

---

## 十一、常见陷阱

### 11.1 忘记 default 导致阻塞

```go
ch := make(chan int)
select {
case v := <-ch:
    fmt.Println(v)
}
// 永久阻塞 → deadlock
```

### 11.2 循环中 time.After 泄漏

```go
// ❌ 每轮创建新 Timer，旧的不回收
for {
    select {
    case <-time.After(time.Second):
    }
}

// ✅ 用 NewTimer + Reset
timer := time.NewTimer(time.Second)
for {
    select {
    case <-timer.C:
        timer.Reset(time.Second)
    }
}
```

### 11.3 select 中 case 数量很多不影响性能

`select` 底层是 O(n) 遍历，n 是 case 数。少量 case 无需担心。

### 11.4 nil channel 的 case 永久阻塞

```go
var ch chan int // nil
select {
case <-ch:      // 永远不会触发
    fmt.Println("never")
case <-time.After(time.Second):
    fmt.Println("timeout")
}
```
这正好可用来**临时禁用**某个 channel，见 6 节示例。

### 11.5 close 已关闭 channel 会 panic

```go
close(ch)
// 别在 select 里 close 已经关过的
```

---

## 十二、select 底层原理（简要）

编译器会把 `select` 转换为运行时调用：

```go
// 伪代码
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool)
```

**流程：**
1. 生成 `scase` 数组，每个 case 描述一个 channel 操作
2. 加锁所有涉及 channel（按地址排序避免死锁）
3. 遍历所有 case，找出就绪的
4. 若多个就绪 → 用 `fastrand` 随机选一个
5. 若都未就绪 → 当前 goroutine 挂起到所有 channel 的等待队列
6. 有 `default` 则直接走 default

**要点：**
- 有 `default`：不挂起，直接返回
- 无 `default`：将当前 goroutine 加入所有相关 channel 的 `sendq`/`recvq`
- 被唤醒后从所有队列移除自己

---

## 十三、实战：完整示例 - 带超时、取消、重试的任务调度

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "time"
)

func fetch(ctx context.Context, id int) (string, error) {
    result := make(chan string, 1)

    go func() {
        // 模拟网络请求
        time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
        result <- fmt.Sprintf("数据-%d", id)
    }()

    select {
    case r := <-result:
        return r, nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    for i := 1; i <= 5; i++ {
        select {
        case <-ctx.Done():
            fmt.Println("整体超时，退出")
            return
        default:
        }

        data, err := fetch(ctx, i)
        if err != nil {
            fmt.Println("任务", i, "失败:", err)
            continue
        }
        fmt.Println("任务", i, "成功:", data)
    }

    fmt.Println("全部完成")
}
```

---

## 十四、select 使用清单

| 场景 | 写法 |
|------|------|
| 等待多个 channel | `select { case <-a: case <-b: }` |
| 非阻塞读取 | 加 `default` |
| 超时控制 | `case <-time.After(d):` |
| 循环超时 | `time.NewTimer` + `Reset` |
| 退出信号 | `case <-done:` 配合 `close(done)` |
| 取消 | `case <-ctx.Done():` |
| 禁用某 case | 把 channel 置 nil |
| 合并多 channel | for + select + `ch = nil` |
| 高优先级 | 嵌套 select |

---

## 十五、最佳实践总结

1. ✅ 循环内用 `select` + `for` 持续监听
2. ✅ 超时优先用 `context.Context`，`time.After` 仅用于单次
3. ✅ 退出用 `close(done)` 广播，不要发送多个值
4. ✅ `ch = nil` 是禁用 case 的惯用技巧
5. ✅ 多 case 随机是特性，不是 bug；要优先级用嵌套 select
6. ❌ 循环里别用 `time.After`，会泄漏定时器
7. ❌ 别在 select 中 close 已经关闭的 channel
8. ❌ 无限循环 + 无 default + 所有 channel 阻塞 → deadlock

---

进一步展开：
- **select 源码解析**（`runtime.selectgo` 的具体实现）
- **context 与 select 配合**的深入用法
- **高并发场景**下 select 的性能优化与替代方案