+++
title = 'Goroutine Channel'
date = '2026-10-12T09:31:48+08:00'
draft = false
categories = ["golang"]
tags = ["goroutine", "channel"]
description = 'Go Channel 详细讲解'

# cover 作为对象配置
[cover]
    image = 'images/cover.svg'           # 图片路径（必填）
    alt = '山高月小，水落石出。'     # 图片替代文本，利于 SEO 和无障碍访问
    caption = '山水' # 图片标题说明（部分主题支持）
    hidden = false                # 是否在单篇文章页隐藏封面？默认 false
    hiddenInList = false          # 是否在文章列表页隐藏封面？默认 false
    hiddenInSingle = false        # 是否在单篇文章页隐藏封面？(部分主题用此字段)
+++


# Go Channel 详细讲解

Channel 是 Go 并发编程的核心，它是 goroutine 之间通信的管道，遵循 **"不要通过共享内存来通信，而要通过通信来共享内存"** 的设计哲学。

---

## 一、Channel 基础概念

### 1.1 什么是 Channel

Channel 是一个**类型化**的管道，可以通过它发送和接收值：

```go
ch := make(chan int)     // 无缓冲 channel
ch <- 42                 // 发送
x := <-ch                // 接收
```

### 1.2 创建方式

```go
// 无缓冲 channel（同步）
ch1 := make(chan int)

// 有缓冲 channel（异步，容量为 3）
ch2 := make(chan string, 3)

// 声明（nil channel，无法使用，会永久阻塞）
var ch3 chan int
```

### 1.3 三种操作

| 操作 | 语法 | 说明 |
|------|------|------|
| 发送 | `ch <- v` | 向 channel 写入 |
| 接收 | `v := <-ch` | 从 channel 读出 |
| 关闭 | `close(ch)` | 关闭 channel |

---

## 二、无缓冲 vs 有缓冲

### 2.1 无缓冲 Channel（同步）

无缓冲 channel 的发送和接收**必须同时准备好**，否则阻塞。这叫"同步通信"。

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan int)

    go func() {
        fmt.Println("准备发送")
        ch <- 100 // 阻塞，直到有人接收
        fmt.Println("发送完成")
    }()

    time.Sleep(time.Second)
    fmt.Println("准备接收")
    v := <-ch
    fmt.Println("接收到:", v)
}
```

**输出：**
```
准备发送
准备接收
接收到: 100
发送完成
```

### 2.2 有缓冲 Channel（异步）

缓冲区未满时可发送，未空时可接收。

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 3)

    ch <- 1
    ch <- 2
    ch <- 3
    fmt.Println("缓冲区:", len(ch), "容量:", cap(ch))

    // ch <- 4  // 会阻塞！缓冲区已满

    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
    fmt.Println(<-ch) // 3
}
```

**关键 API：**
- `len(ch)`：当前缓冲的元素个数
- `cap(ch)`：缓冲区容量

---

## 三、关闭 Channel 与 range 遍历

### 3.1 关闭 Channel

```go
close(ch)
```

**规则：**
- 只能由**发送方**关闭
- 关闭后不能再发送（panic）
- 关闭后仍可接收剩余数据
- 对已关闭 channel 再关闭会 panic

### 3.2 接收的两个返回值

```go
v, ok := <-ch
// ok == true  : 正常接收
// ok == false : channel 已关闭且无数据
```

### 3.3 range 遍历

`range` 会自动检测 channel 关闭，收到所有数据后退出循环。

```go
package main

import "fmt"

func producer(ch chan int) {
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch) // 必须关闭，否则 range 会死锁
}

func main() {
    ch := make(chan int)
    go producer(ch)

    for v := range ch {
        fmt.Println("收到:", v)
    }
    fmt.Println("channel 已关闭")
}
```

---

## 四、单向 Channel

用于限制 channel 的使用方向，提高类型安全。

```go
chan int    // 双向
chan<- int  // 只写
<-chan int  // 只读
```

```go
package main

import "fmt"

// 只写
func send(ch chan<- int) {
    for i := 0; i < 3; i++ {
        ch <- i
    }
    close(ch)
}

// 只读
func receive(ch <-chan int) {
    for v := range ch {
        fmt.Println(v)
    }
}

func main() {
    ch := make(chan int)
    go send(ch)
    receive(ch)
}
```

> 双向 channel 可隐式转换为单向，反之不行。

---

## 五、select 多路复用

`select` 让 goroutine 同时等待多个 channel 操作。

### 5.1 基础用法

```go
select {
case v := <-ch1:
    fmt.Println("来自 ch1:", v)
case v := <-ch2:
    fmt.Println("来自 ch2:", v)
case ch3 <- 10:
    fmt.Println("写入 ch3")
default:
    fmt.Println("无就绪操作")
}
```

### 5.2 超时控制

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
        fmt.Println("超时了")
    }
}
```

### 5.3 非阻塞读取

```go
select {
case v := <-ch:
    fmt.Println("收到:", v)
default:
    fmt.Println("没有数据")
}
```

---

## 六、常见模式

### 6.1 生产者-消费者

```go
package main

import (
    "fmt"
    "sync"
)

func producer(id int, ch chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 3; i++ {
        ch <- id*10 + i
    }
}

func main() {
    ch := make(chan int, 10)
    var wg sync.WaitGroup

    // 3 个生产者
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go producer(i, ch, &wg)
    }

    // 等待所有生产者后关闭
    go func() {
        wg.Wait()
        close(ch)
    }()

    // 消费者
    for v := range ch {
        fmt.Println("消费:", v)
    }
}
```

### 6.2 Worker Pool（工作池）

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for j := range jobs {
        results <- j * j
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    var wg sync.WaitGroup
    // 启动 3 个 worker
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // 发送任务
    for i := 1; i <= 9; i++ {
        jobs <- i
    }
    close(jobs)

    // 等 worker 结束再关闭结果
    go func() {
        wg.Wait()
        close(results)
    }()

    for r := range results {
        fmt.Println("结果:", r)
    }
}
```

### 6.3 扇入扇出（Fan-in / Fan-out）

```go
// Fan-out：多个 goroutine 消费同一个 channel
// Fan-in：多个 channel 合并到一个

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

### 6.4 用 channel 实现信号量

```go
sem := make(chan struct{}, 3) // 最多 3 个并发

for i := 0; i < 10; i++ {
    sem <- struct{}{}  // 获取
    go func(i int) {
        defer func() { <-sem }() // 释放
        // ... 业务逻辑
    }(i)
}
```

---

## 七、常见陷阱

### 7.1 死锁

```go
func main() {
    ch := make(chan int)
    ch <- 1  // 死锁！没人接收
}
```

**报错：** `fatal error: all goroutines are asleep - deadlock!`

### 7.2 向已关闭 channel 发送

```go
ch := make(chan int)
close(ch)
ch <- 1  // panic: send on closed channel
```

### 7.3 重复关闭

```go
ch := make(chan int)
close(ch)
close(ch)  // panic: close of closed channel
```

### 7.4 nil channel

对 nil channel 的发送和接收会**永久阻塞**：

```go
var ch chan int
<-ch      // 永久阻塞
ch <- 1   // 永久阻塞
close(ch) // panic
```

> 💡 `select` 中可用 nil channel 来"禁用"某个 case。

### 7.5 goroutine 泄漏

```go
func bad() {
    ch := make(chan int)
    go func() {
        ch <- 1 // 如果没人接收，goroutine 永久阻塞
    }()
    // 忘了接收
}
```

**解决：** 使用缓冲、使用 context 取消、确保有接收方。

---

## 八、Channel 底层原理（简要）

```go
type hchan struct {
    qcount   uint           // 当前元素数量
    dataqsiz uint           // 缓冲区大小
    buf      unsafe.Pointer // 环形缓冲区
    elemsize uint16
    closed   uint32
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 等待接收的 goroutine 队列
    sendq    waitq          // 等待发送的 goroutine 队列
    lock     mutex
}
```

**要点：**
1. 有缓冲 channel 底层是**环形队列**
2. 等待的 goroutine 被挂到 `recvq` / `sendq`
3. 无缓冲发送时，直接把数据从 sender 拷贝到 receiver（"接力"）
4. 所有操作都需要**加锁**，因此高频通信可考虑其他方式

---

## 九、Channel vs 其他同步机制

| 场景 | 推荐 |
|------|------|
| goroutine 间传数据 | Channel |
| 只做互斥保护 | `sync.Mutex` |
| 等待多个任务完成 | `sync.WaitGroup` |
| 一次性通知 | `close(ch)` 或 `sync.Once` |
| 状态共享、缓存 | `sync.Map` / 原子操作 |

**性能提示：**
- 无缓冲 channel ≈ 一次 goroutine 切换成本
- 有缓冲 channel 更快（减少阻塞）
- 极高吞吐场景可考虑 `sync/atomic` 或分片锁

---

## 十、最佳实践总结

1. ✅ **谁发送，谁关闭**（谁负责 close）
2. ✅ 用 `range` 或 `v, ok := <-ch` 安全接收
3. ✅ 使用 `select` + `time.After` 做超时
4. ✅ 用 `context.Context` 处理取消信号
5. ✅ Worker Pool 用 `WaitGroup` + `close` 组合
6. ❌ 不要向已关闭的 channel 发送
7. ❌ 不要依赖 channel 做广播（用 `close` 或 context）
8. ❌ 不要用 channel 代替简单的互斥锁

---