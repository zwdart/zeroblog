---
title: 'Golang Profile'
date: '2026-11-05T11:35:50+08:00'
categories: [golang,Profile]
tags: [Profile]
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


Go 的 **profile** 是一套内置于 runtime 的性能诊断机制，它通过**采样或事件记录**的方式，持续收集程序运行时的 CPU 使用、内存分配、Goroutine 状态、锁竞争等数据，最终生成一个可分析的数据文件（`.prof`）。`go tool pprof` 则是解读这些数据的官方工具，能将原始数据转化为调用图、火焰图或源码级别的热点标注，帮你定位性能瓶颈。

---

### 📊 Profile 的核心类型：按“症状”选择工具

每种 Profile 回答不同的问题，选错类型会浪费大量调查时间。

| Profile 类型 | 采集标志 / HTTP 端点 | 核心用途 | 关键说明 |
| :--- | :--- | :--- | :--- |
| **CPU** | `-cpuprofile` 或 `/debug/pprof/profile?seconds=30` | 高 CPU 占用、函数执行慢 | 以 **100Hz** 采样**在 CPU 上**运行的函数；**无法捕获** I/O、Sleep 等 Off-CPU 时间 |
| **Heap (内存)** | `-memprofile` 或 `/debug/pprof/heap` | GC 压力大、内存泄漏 | 有四种视角：`alloc_objects`（分配次数）、`alloc_space`（分配字节数）、`inuse_space`（当前存活字节）、`inuse_objects`（当前存活对象数） |
| **Goroutine** | `/debug/pprof/goroutine` | Goroutine 泄漏、阻塞堆积 | 对所有 Goroutine 的调用栈进行快照；查找大量 Goroutine 堆积在**同一调用点**的情况 |
| **Mutex** | `/debug/pprof/mutex` | 锁竞争激烈 | 记录 Goroutine 等待获取锁的**累计时间**；默认关闭，需 `runtime.SetMutexProfileFraction()` 开启 |
| **Block** | `/debug/pprof/block` | Channel/Mutex/Select 阻塞 | 记录 Goroutine 在同步原语上的**累计阻塞时间**；默认关闭，需 `runtime.SetBlockProfileRate()` 开启 |
| **Threadcreate** | `/debug/pprof/threadcreate` | OS 线程过度创建 | 显示创建新 OS 线程的调用栈，通常由 cgo 或阻塞系统调用导致 |

---

### 🛠️ 三种采集方式：按“场景”选择入口

#### 💻 代码植入式（`runtime/pprof`）：适用于独立程序或精确控制

在代码中手动启动和停止，适合命令行工具、离线分析或需要精确控制采集时机的场景。

```go
package main

import (
	"os"
	"runtime/pprof"
	"time"
)

func main() {
	// 启动 CPU Profile
	f, _ := os.Create("cpu.prof")
	pprof.StartCPUProfile(f)
	defer pprof.StopCPUProfile() // 确保程序退出前停止并写入文件

	// 模拟耗时工作
	time.Sleep(2 * time.Second)
	busyWork()

	// 在特定时间点采集 Heap Profile
	hf, _ := os.Create("heap.prof")
	pprof.WriteHeapProfile(hf) // 快照当前堆状态
	hf.Close()
}

func busyWork() {
	for i := 0; i < 1e8; i++ {
		_ = i * i
	}
}
```

**运行与分析**：`go run main.go` 后会生成 `cpu.prof` 和 `heap.prof`，然后执行 `go tool pprof cpu.prof` 进入交互界面。

#### 🌐 HTTP 端点式（`net/http/pprof`）：适用于在线服务（最常用）

为运行中的服务开一个“诊断端口”，无需重启即可实时采集，是生产环境排查问题的首选。

```go
package main

import (
	"log"
	"net/http"
	_ "net/http/pprof" // 匿名导入，自动注册 /debug/pprof/ 路由
	"time"
)

func main() {
	// 在独立端口启动 pprof 服务，避免与业务端口冲突
	go func() {
		log.Println(http.ListenAndServe(":6060", nil))
	}()

	// 业务逻辑（模拟）
	for {
		time.Sleep(time.Second)
	}
}
```

**运行与分析**：

1.  `go run main.go` 启动服务。
2.  新终端执行采集：
    - **CPU**：`go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30`（采集 30 秒）
    - **Heap**：`go tool pprof http://localhost:6060/debug/pprof/heap`
    - **Goroutine**：`go tool pprof http://localhost:6060/debug/pprof/goroutine`

#### 🧪 基准测试式（`go test`）：适用于微基准和优化对比

在基准测试中直接采集，无需编写额外代码，适合对比优化前后的差异。

```go
// 在 _test.go 文件中
func BenchmarkBusyWork(b *testing.B) {
	for i := 0; i < b.N; i++ {
		busyWork()
	}
}
```

**运行与分析**：

```bash
go test -bench=BenchmarkBusyWork -cpuprofile=cpu.prof -memprofile=mem.prof .
go tool pprof -http=:8080 cpu.prof   # 直接打开 Web UI 查看
```

---

### 🔍 分析工具：从数据到洞察

#### 💻 交互式 CLI：`go tool pprof`

进入交互模式后，核心命令如下：

| 命令 | 作用 |
| :--- | :--- |
| **`top`** | 按**自身耗时**排名，首选命令 |
| **`top -cum`** | 按**累积耗时**排名（包含子调用） |
| **`list funcName`** | 展示函数的**源码级**耗时标注 |
| **`web`** | 生成**调用图**（需 Graphviz） |
| **`peek funcName`** | 查看函数的调用者和被调用者 |

#### 🌐 Web UI：`-http` 标志（推荐）

`go tool pprof -http=:8080 cpu.prof` 会在浏览器打开交互式界面，核心视图是**火焰图（Flame Graph）**。

**火焰图读法**：
- **纵轴**：调用栈深度，越往下是越底层的调用。
- **横轴**：代表耗时占比，**越宽表示该函数消耗的时间越多**。
- **交互**：鼠标悬浮可查看详细数据，点击可放大该分支。

#### 🔥 火焰图生成工具：`go-torch`（经典工具）

Uber 开源的 `go-torch` 可直接从运行中的服务或 `.prof` 文件生成交互式 SVG 火焰图。

```bash
# 安装
go install github.com/uber/go-torch@latest

# 从运行中的服务生成（采集 30 秒）
go-torch -u http://localhost:6060 -t 30 -f profile.svg

# 从已有的 .prof 文件生成
go-torch --binaryinput cpu.prof --binaryname <你的二进制文件名>
```

生成的 `profile.svg` 可用浏览器打开，交互体验与 `pprof -http` 的火焰图类似。

---

### 💎 总结

Go 的 Profile 体系可以概括为 **“三类入口、七种视图、一个工具”**：

- **三类入口**：代码植入（`runtime/pprof`）、HTTP 端点（`net/http/pprof`）、基准测试（`go test`）。
- **七种视图**：CPU、Heap（四种子视图）、Goroutine、Mutex、Block、Threadcreate，**按症状选择**。
- **一个工具**：`go tool pprof`，通过 CLI 或 Web UI（火焰图）将数据转化为可操作的优化线索。

**核心原则**：**先匹配症状到 Profile 类型，再选择采集入口，最后用火焰图定位热点**。Mutex 和 Block Profile 默认关闭，需要时务必先通过 `runtime` 包开启。