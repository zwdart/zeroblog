---
title: Goroutine详细：sync.WaitGroup
date: 2026-10-11T22:51:47+08:00
lastmod: 2026-10-11T22:52:47+08:00
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
  - WaitGroup
# nolastmod: true
# math: true
draft: false
---

# sync.WaitGroup 是什么.

`sync.WaitGroup` 是 Go 标准库中用于**等待一组 goroutine 完成**的同步原语。它解决的核心问题是：主 goroutine 需要知道其他多个 goroutine 何时全部执行完毕。

## 基本用法

```go
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    wg.Add(1)          // 计数 +1
    go func(id int) {
        defer wg.Done() // 计数 -1
        // 做一些工作
    }(i)
}

wg.Wait() // 阻塞，直到计数归零
```

## 为什么需要 Add 和 Done

WaitGroup 内部维护一个**计数器**，它的语义是"还有多少个任务没完成"：

- **`Add(delta)`**：告诉 WaitGroup "我要开始 `delta` 个任务了"，计数器增加。
- **`Done()`**：等价于 `Add(-1)`，表示"我完成了 1 个任务"，计数器减少。
- **`Wait()`**：阻塞当前 goroutine，直到计数器归零。

拆成两个方法而不是一个"注册+运行"的 API，是因为**任务的启动和完成在时间上是分离的**：

1. 你需要在**启动 goroutine 之前**就 `Add`，否则可能出现 goroutine 已经跑完并 `Done` 了、但 `Add` 还没执行的竞态，导致计数器错乱甚至 `Wait` 永远不返回。
2. `Done` 必须在**任务真正结束时**调用，通常配合 `defer`，保证即使 panic 或提前 return 也能正确减计数。

## 关键细节

**Add 必须在 Wait 之前、且在 goroutine 启动前调用**

```go
// 错误：可能在 Wait 已返回后才 Add
go func() {
    wg.Add(1)  // 竞态！
    defer wg.Done()
    // ...
}()
wg.Wait()
```

正确写法是在 `go` 语句之前 `wg.Add(1)`。

**计数器不能变成负数**：`Done` 调用次数多于 `Add` 会 panic。

**WaitGroup 不可复制**：它内部含 `noCopy` 标记，传参要用指针 `*sync.WaitGroup`。

**复用**：等 `Wait` 返回、计数器归零后，可以再次 `Add` 复用同一个 WaitGroup（但要注意此时不能有并发的 Wait）。

## 和其他原语的区别

| 原语 | 用途 |
|------|------|
| `sync.WaitGroup` | 等待一组任务全部完成 |
| `sync.Mutex` | 互斥访问共享资源 |
| channel | 更通用的通信/同步，可实现类似 WaitGroup 的效果但更灵活 |

简单说，Add/Done 的设计是为了把"声明任务数量"和"标记任务结束"这两个时机分开，既保证计数正确，又让完成通知可以自然地用 `defer` 表达。