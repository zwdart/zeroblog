---
title: 'Template text/template 完全学习'
date: '2026-10-13T11:05:20+08:00'
categories: ['golang']
tags: ['goroutine','template']
description: 'text/template 专门用于生成**任意文本输出**（HTML、配置文件、代码、Prompt 等）'
draft: true

cover:
  image: 'images/cover.svg'
  alt: '山高月小，水落石出。'
  caption: '山水'
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
---



# Go 语言 `text/template` 完全指南（含可运行示例）

`text/template` 是 Go 标准库自带的模板引擎，专门用于生成**任意文本输出**（HTML、配置文件、代码、Prompt 等）。下面从零讲到进阶，每个知识点都配可运行代码。

---

## 一、为什么需要模板引擎？

假设你要生成一段 Prompt：

```go
// 字符串拼接
prompt := "你是一个" + role + "助手。请用" + style + "风格回答。"
```

一旦需求变复杂——要根据条件加 few-shot 示例、要循环生成示例、要做 A/B 实验——字符串拼接就变得难以维护。

模板引擎把"文案"和"逻辑"分离：

```
你是一个{{ .Role }}助手。请用{{ .Style }}风格回答。
```

代码只负责提供数据，模板负责渲染。

---

## 二、最小可运行示例

### `main.go`

```go
package main

import (
	"os"
	"text/template"
)

func main() {
	tmpl := template.Must(template.New("hello").Parse("你好，{{ .Name }}！你今年 {{ .Age }} 岁。\n"))

	data := map[string]any{
		"Name": "小明",
		"Age":  18,
	}

	if err := tmpl.Execute(os.Stdout, data); err != nil {
		panic(err)
	}
}
```

运行：

```bash
go run main.go
```

输出：

```
你好，小明！你今年 18 岁。
```

这就是模板的核心三步：**Parse → Execute → 输出**。

---

## 三、三个核心概念

| 概念 | 说明 |
| :--- | :--- |
| **Template** | 解析后的模板对象，可重复使用 |
| **Data** | 传给模板的数据，可以是 map、struct、基本类型 |
| **Execute** | 把模板与数据结合，写入 `io.Writer` |

关键点：

- `Parse` 只做一次，`Execute` 可以反复调用（并发安全）。
- `Execute` 的第二个参数就是模板里的 `.`（点）。

---

## 四、模板语法详解

### 1. 输出变量 `{{ .字段 }}`

```go
tmpl := template.Must(template.New("x").Parse("{{ .Name }} 在 {{ .City }}\n"))
data := map[string]string{"Name": "小明", "City": "杭州"}
tmpl.Execute(os.Stdout, data)
// 输出：小明 在 杭州
```

### 2. struct 作为数据

```go
type User struct {
	Name string
	Age  int
}

tmpl := template.Must(template.New("x").Parse("{{ .Name }} 今年 {{ .Age }} 岁\n"))
tmpl.Execute(os.Stdout, User{Name: "小明", Age: 18})
// 输出：小明 今年 18 岁
```

> **推荐用 struct**：字段名写错会在编译期报错，比 map 更安全。

### 3. 条件判断 `{{ if }}` `{{ else }}` `{{ end }}`

```go
const t = `{{ if .IsVIP }}尊贵的 VIP 用户{{ else }}普通用户{{ end }}，你好！
`
tmpl := template.Must(template.New("x").Parse(t))

tmpl.Execute(os.Stdout, map[string]bool{"IsVIP": true})
tmpl.Execute(os.Stdout, map[string]bool{"IsVIP": false})
```

输出：

```
尊贵的 VIP 用户，你好！
普通用户，你好！
```

`if` 会判断值是否为"零值"：`false`、`0`、`""`、`nil`、空切片都是假。

### 4. 循环 `{{ range }}`

```go
const t = `示例：
{{ range .Examples }}输入：{{ .Input }}
输出：{{ .Output }}
{{ end }}`

data := map[string]any{
	"Examples": []map[string]string{
		{"Input": "这家餐厅很棒", "Output": "正面"},
		{"Input": "太失望了", "Output": "负面"},
		{"Input": "今天天气不错", "Output": "中性"},
	},
}

tmpl := template.Must(template.New("x").Parse(t))
tmpl.Execute(os.Stdout, data)
```

输出：

```
示例：
输入：这家餐厅很棒
输出：正面
输入：太失望了
输出：负面
输入：今天天气不错
输出：中性
```

`range` 里 `.` 变成当前元素，所以 `{{ .Input }}` 取当前元素的字段。

### 5. 循环里的索引

```go
const t = `{{ range $i, $v := .Items }}{{ $i }}: {{ $v }}
{{ end }}`

tmpl := template.Must(template.New("x").Parse(t))
tmpl.Execute(os.Stdout, map[string]any{"Items": []string{"a", "b", "c"}})
```

输出：

```
0: a
1: b
2: c
```

### 6. 变量 `{{ $x := ... }}`

```go
const t = `{{ $name := .Name }}你好，{{ $name }}！{{ $name }}，欢迎回来。
`
tmpl := template.Must(template.New("x").Parse(t))
tmpl.Execute(os.Stdout, map[string]string{"Name": "小明"})
```

输出：

```
你好，小明！小明，欢迎回来。
```

变量在 `{{ end }}` 后失效，也可以在 `range` 外定义、内部使用。

### 7. 管道 `{{ . | func }}`

```go
const t = `{{ .Name | printf "%s先生" }}{{ .Text | len }} 个字
`
tmpl := template.Must(template.New("x").Parse(t))
tmpl.Execute(os.Stdout, map[string]string{"Name": "小明", "Text": "你好世界"})
```

输出：

```
小明先生4 个字
```

管道把前一个结果作为后一个函数的最后一个参数。

### 8. 内建函数一览

| 函数 | 作用 | 示例 |
| :--- | :--- | :--- |
| `len` | 长度 | `{{ .Text \| len }}` |
| `index` | 索引 | `{{ index .Items 0 }}` |
| `printf` | 格式化 | `{{ printf "%d" .Age }}` |
| `print` / `println` | 打印 | `{{ print .Name }}` |
| `eq` / `ne` | 等于/不等于 | `{{ if eq .A .B }}` |
| `lt` / `le` / `gt` / `ge` | 比较 | `{{ if gt .Age 18 }}` |
| `and` / `or` / `not` | 逻辑 | `{{ if and .A .B }}` |
| `urlquery` | URL 转义 | `{{ .Q \| urlquery }}` |
| `html` | HTML 转义 | `{{ .T \| html }}` |
| `js` | JS 转义 | `{{ .T \| js }}` |

### 9. 去除空白 `{{-` 和 `-}}`

模板里的换行和缩进会原样输出。用 `-` 可以吃掉两侧空白：

```go
const t = `示例：
{{- range .Examples }}
输入：{{ .Input }}
{{- end }}
`
```

输出会去掉 `range` 和 `end` 行产生的空行。

### 10. 注释 `{{/* ... */}}`

```go
const t = `{{/* 这是注释，不会输出 */}}你好，{{ .Name }}
`
```

---

## 五、多模板与嵌套

### 1. 定义命名模板 `{{ define }}`

```go
const t = `
{{ define "greeting" }}你好，{{ .Name }}！{{ end }}
{{ define "body" }}欢迎使用本系统。{{ end }}

{{ template "greeting" . }}
{{ template "body" . }}
`

tmpl := template.Must(template.New("x").Parse(t))
tmpl.Execute(os.Stdout, map[string]string{"Name": "小明"})
```

输出：

```
你好，小明！
欢迎使用本系统。
```

### 2. 模板里调用模板

```go
const t = `
{{ define "example" }}输入：{{ .Input }}
输出：{{ .Output }}
{{ end }}

示例：
{{ range .Examples }}{{ template "example" . }}{{ end }}
`

data := map[string]any{
	"Examples": []map[string]string{
		{"Input": "很好", "Output": "正面"},
		{"Input": "太差", "Output": "负面"},
	},
}

tmpl := template.Must(template.New("x").Parse(t))
tmpl.Execute(os.Stdout, data)
```

输出：

```
示例：
输入：很好
输出：正面
输入：太差
输出：负面
```

---

## 六、从文件加载模板

真实项目里模板通常放在文件里。

### 目录结构

```
.
├── main.go
└── tmpl/
    ├── greeting.tmpl
    └── example.tmpl
```

### `tmpl/greeting.tmpl`

```gotemplate
你好，{{ .Name }}！欢迎使用。
```

### `tmpl/example.tmpl`

```gotemplate
示例：
{{- range .Examples }}
输入：{{ .Input }}
输出：{{ .Output }}
{{- end }}
```

### `main.go`

```go
package main

import (
	"os"
	"text/template"
)

func main() {
	// 加载单个文件
	t1 := template.Must(template.ParseFiles("tmpl/greeting.tmpl"))
	t1.Execute(os.Stdout, map[string]string{"Name": "小明"})

	// 加载多个文件
	t2 := template.Must(template.ParseGlob("tmpl/*.tmpl"))
	// 指定用哪个模板渲染
	t2.ExecuteTemplate(os.Stdout, "example.tmpl", map[string]any{
		"Examples": []map[string]string{
			{"Input": "很好", "Output": "正面"},
			{"Input": "太差", "Output": "负面"},
		},
	})
}
```

**关键点**：用 `ParseFiles` / `ParseGlob` 加载多文件时，模板名就是文件名。渲染时用 `ExecuteTemplate(w, "文件名", data)` 指定用哪个。

---

## 七、自定义函数

模板可以注册自定义函数，扩展表达能力。

```go
package main

import (
	"os"
	"strings"
	"text/template"
)

func main() {
	funcs := template.FuncMap{
		"upper": strings.ToUpper,
		"trunc": func(n int, s string) string {
			if len(s) <= n {
				return s
			}
			return s[:n] + "..."
		},
		"add": func(a, b int) int { return a + b },
	}

	const t = `{{ .Name | upper }} 
{{ .Text | trunc 10 }}
1 + 2 = {{ add 1 2 }}
`

	tmpl := template.Must(template.New("x").Funcs(funcs).Parse(t))

	tmpl.Execute(os.Stdout, map[string]string{
		"Name": "xiaoming",
		"Text": "这是一段很长的文本内容需要截断",
	})
}
```

输出：

```
XIAOMING 
这是一段很长的文...
1 + 2 = 3
```

**注意**：

- 自定义函数必须在 `Parse` 之前用 `Funcs` 注册。
- 函数只能返回一个值或 `(值, error)`；返回 error 时，模板执行会中止并报错。
- 函数名是模板里用的名字，可以和 Go 函数名不同。

---

## 八、错误处理

`Parse` 和 `Execute` 都可能出错，生产代码必须处理。

```go
tmpl, err := template.New("x").Parse("{{ .NotExist }}")
if err != nil {
	// 解析错误：语法问题
	log.Fatal(err)
}

err = tmpl.Execute(os.Stdout, map[string]string{"Name": "x"})
if err != nil {
	// 执行错误：字段不存在
	log.Fatal(err)
}
```

用 `template.Must` 可以在出错时直接 panic，适合初始化阶段：

```go
tmpl := template.Must(template.New("x").Parse("..."))
```

**规则**：启动时加载模板用 `Must`，运行时 `Execute` 用 `if err != nil`。

---

## 九、`text/template` vs `html/template`

| 维度 | `text/template` | `html/template` |
| :--- | :--- | :--- |
| 用途 | 纯文本、代码、配置文件、Prompt | 生成 HTML |
| 输出转义 | 不转义 | 自动 HTML 转义 |
| 防 XSS | 不负责 | 自动防 XSS |
| 函数集 | 通用 | 增加 HTML 相关函数 |
| 用法 | 基本相同 | 基本相同 |

**规则**：

- 生成 HTML → 用 `html/template`
- 生成 Prompt、配置文件、代码 → 用 `text/template`

它们的模板语法完全一致，只是转义策略不同。

---

## 十、完整可运行示例：Prompt 模板引擎

把上面所有知识点串起来，做一个 Prompt 工程里常见的例子。

### 目录结构

```
.
├── main.go
├── config.json
└── prompts/
    └── classify.tmpl
```

### `config.json`

```json
{
  "role": "文本情感分类助手",
  "labels": ["正面", "负面", "中性"],
  "style": "简洁",
  "examples": [
    {"Input": "这家餐厅很棒", "Output": "正面"},
    {"Input": "太失望了", "Output": "负面"},
    {"Input": "今天天气不错", "Output": "中性"}
  ]
}
```

### `prompts/classify.tmpl`

```gotemplate
你是一个{{ .Role }}。请对用户输入的文本进行分类。

{{- if .Labels }}
可选标签：{{ range $i, $l := .Labels }}{{ if $i }}、{{ end }}{{ $l }}{{ end }}
{{- end }}

风格要求：{{ .Style }}

{{- if .Examples }}
示例：
{{- range .Examples }}
输入：{{ .Input }}
输出：{{ .Output }}
{{- end }}
{{- end }}

输入：{{ .Input }}
输出：
```

### `main.go`

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"strings"
	"text/template"
)

type Example struct {
	Input  string `json:"input"`
	Output string `json:"output"`
}

type Config struct {
	Role     string    `json:"role"`
	Labels   []string  `json:"labels"`
	Style    string    `json:"style"`
	Examples []Example `json:"examples"`
}

type PromptData struct {
	Role     string
	Labels   []string
	Style    string
	Examples []Example
	Input    string
}

func main() {
	// 1. 读取配置
	cfgBytes, err := os.ReadFile("config.json")
	if err != nil {
		panic(err)
	}
	var cfg Config
	if err := json.Unmarshal(cfgBytes, &cfg); err != nil {
		panic(err)
	}

	// 2. 加载模板
	tmpl := template.Must(template.ParseFiles("prompts/classify.tmpl"))

	// 3. 渲染
	data := PromptData{
		Role:     cfg.Role,
		Labels:   cfg.Labels,
		Style:    cfg.Style,
		Examples: cfg.Examples,
		Input:    "Go 语言的并发模型非常优雅，我很喜欢。",
	}

	var sb strings.Builder
	if err := tmpl.Execute(&sb, data); err != nil {
		panic(err)
	}

	fmt.Println(sb.String())
}
```

### 运行

```bash
go run main.go
```

### 输出

```
你是一个文本情感分类助手。请对用户输入的文本进行分类。

可选标签：正面、负面、中性

风格要求：简洁

示例：
输入：这家餐厅很棒
输出：正面
输入：太失望了
输出：负面
输入：今天天气不错
输出：中性

输入：Go 语言的并发模型非常优雅，我很喜欢。
输出：
```

这个例子覆盖了：

- struct 作为数据
- `if` 条件
- `range` 循环
- 嵌套 `if` + `range`
- `$i` 索引变量
- 空白控制 `{{-` / `-}}`
- 从文件加载模板

---

## 十一、常见坑与最佳实践

| 坑 | 说明 | 建议 |
| :--- | :--- | :--- |
| 字段不存在 | `{{ .NotExist }}` 执行时报错 | 用 struct 而非 map，编译期发现拼写 |
| 空白太多 | 换行缩进原样输出 | 用 `{{-` 和 `-}}` 控制 |
| `Parse` 只做一次 | 循环里重复 Parse 很浪费 | Parse 一次，Execute 多次 |
| 并发安全 | `Execute` 是并发安全的 | 一个 tmpl 可被多 goroutine 复用 |
| 自定义函数顺序 | 必须 Parse 前注册 | `New().Funcs(...).Parse(...)` |
| HTML 场景误用 | `text/template` 不防 XSS | HTML 输出改用 `html/template` |
| 模板名混淆 | `ParseFiles` 时模板名=文件名 | 用 `ExecuteTemplate` 指定 |

---

## 十二、函数速查表

| 函数 | 说明 |
| :--- | :--- |
| `and` `or` `not` | 逻辑运算 |
| `eq` `ne` `lt` `le` `gt` `ge` | 比较 |
| `len` | 长度 |
| `index` | 索引 |
| `slice` | 切片 |
| `print` `printf` `println` | 格式化 |
| `urlquery` | URL 查询转义 |
| `html` `js` `urlquery` | 转义 |
| `call` | 调用函数值 |

---

## 十三、一句话总结

> **`text/template` 用 `Parse` 解析模板、`Execute` 渲染数据，支持变量、条件、循环、管道、嵌套、自定义函数，把"文案"与"逻辑"分离。它语法简单、标准库自带、无依赖，是 Go 生成任意文本（包括 Prompt）的首选。**

上面的完整示例可以直接 `go run`，覆盖了日常使用 90% 的场景。需要更复杂的场景（如热加载、多模板继承）时，再叠加 `ParseGlob` 和 `define` / `template` 即可。