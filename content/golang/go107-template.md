---
title: 'Template 系统性学习'
date: '2026-10-13T11:05:20+08:00'
categories: ['golang']
tags: ['goroutine','template']
description: '服务端渲染和数据驱动的文本生成工具'
draft: false

cover:
  image: 'images/cover.svg'
  alt: '山高月小，水落石出。'
  caption: '山水'
  hidden: false
  hiddenInList: false
  hiddenInSingle: false
---


# Go Template 系统性学习指南

Go 标准库的模板引擎是服务端渲染和数据驱动的文本生成工具，核心设计理念是**将表现层与数据模型分离**。Go 用同一套语法提供了两个包：`text/template` 用于通用文本生成，`html/template` 用于 HTML 输出并自带上下文感知的自动转义安全机制。网络服务中生成 HTML 必须使用 `html/template`。

## 一、基础用法：三步流程

模板使用的核心流程固定为三步：**定义/解析模板 → 准备数据 → 执行渲染**。

### 1.1 从字符串解析

```go
package main

import (
    "os"
    "text/template"
)

type Inventory struct {
    Material string
    Count    uint
}

func main() {
    // 步骤1：解析模板
    tmpl := template.Must(template.New("test").
        Parse("{{.Count}} items are made of {{.Material}}"))

    // 步骤2：准备数据
    sweaters := Inventory{"wool", 17}

    // 步骤3：执行渲染
    tmpl.Execute(os.Stdout, sweaters)
    // 输出：17 items are made of wool
}
```

`template.Must` 是对 `Parse` 错误的 panic 包装，适合在包初始化阶段使用。**模板中只能访问结构体的导出字段（首字母大写）**，小写字段在模板侧不可达，会导致执行期报错或呈现零值。

### 1.2 从文件解析

```go
// 解析单个文件
t, err := template.ParseFiles("templates/layout.html")

// 解析多个文件（常用于主模板+子模板）
t, err := template.ParseFiles("templates/layout.html", "templates/header.html")

// 使用 Glob 模式批量解析
t, err := template.ParseGlob("templates/*.html")
```

### 1.3 数据访问方式

```go
// 结构体字段访问
t.Execute(w, struct{ Name string }{"Alice"})  // {{.Name}}

// map 键访问（无大小写限制）
t.Execute(w, map[string]string{"name": "Bob"}) // {{.name}} 或 {{.Name}} 均可

// 点（.）表示当前作用域的值
// 在根模板中，{{.}} 就是 Execute 传入的整个数据
```

## 二、模板语法核心

### 2.1 注释

```go
{{/* 这是注释，不会输出 */}}

{{- /* 注释，同时修剪前后空白 */ -}}
```

### 2.2 条件判断

Go 模板中，以下值被视为“假”：`false`、`0`、`nil`、空字符串、空切片/映射/数组。

```go
{{if .Attended}}
    很高兴您能参加婚礼。
{{else}}
    很遗憾您未能参加。
{{end}}
```

`if` 支持 `else if`，也常与内置比较函数配合：

```go
{{if eq .Status "active"}} 活跃 {{else if gt .Score 90}} 优秀 {{end}}
```

常用比较函数：`eq`（等于）、`ne`（不等于）、`lt`（小于）、`gt`（大于）、`le`（小于等于）、`ge`（大于等于）。

### 2.3 循环 range

`range` 支持 slice、array、map、channel。内部 `{{.}}` 为当前元素；可通过变量捕获索引/键和值。

```go
// 遍历切片：下标 + 值
{{range $idx, $email := .Emails}}
    第 {{$idx}} 项: {{$email}}
{{end}}

// 遍历 map：key + value
{{range $key, $value := .MapContent}}
    {{$key}}: {{$value}}
{{end}}
```

`range` 在空集合时不执行，`. ` 不会改变。若只需值，使用 `{{range .Items}}` 即可。

### 2.4 with 改变作用域

`with` 将点临时切换到某个值，适合简化深层字段访问：

```go
{{with .Gift}}
    感谢您送的 {{.}}。
{{end}}
```

若 `.Gift` 为“假值”，`with` 块不执行。

### 2.5 变量

```go
{{$v := .SomeValue}}      {{/* 声明并赋值 */}}
{{$v = "new value"}}      {{/* 重新赋值（Go 1.11+） */}}
```

变量在 `range` 和 `with` 改变点作用域后仍然可读，这是保留根数据引用的常用手段。

### 2.6 管道

管道语义类似 UNIX：`{{ a | f | g }}` 等价于 `g(f(a))`。前一个函数的结果作为下一个函数的**最后一个参数**。

```go
{{ .Personal | printf "[%s]" }}
{{ .Name | upper | printf "用户: %s" }}
```

管道让展示层的轻量格式化（如 `printf`）保持在模板内，避免把“仅为排版服务”的逻辑塞进业务层。

### 2.7 空白修剪

模板中的换行和缩进会原样输出，这在生成代码或结构化文本时往往需要精确控制。

- `{{-` ：修剪**左侧**紧邻文本的尾部空白
- `-}}` ：修剪**右侧**紧邻文本的前导空白

```go
// 不修剪时，range 循环会引入多余换行
{{range .Items}}
    {{.}}
{{end}}

// 双向修剪，保持紧凑输出
{{- range .Items -}}
    {{.}}
{{- end -}}
```

注意：`{{- 3}}` 修剪左侧空白后解析为数字 `3`；而 `{{-3}}` 会被解析为**负数 -3** 的操作。修剪符后的空格必须存在。

## 三、自定义函数

`template.FuncMap` 用于将 Go 函数注册到模板命名空间中。**必须先调用 `Funcs()` 注册函数，再调用 `Parse()`**，否则 AST 中出现的未知标识符会导致 Parse 失败。

```go
funcMap := template.FuncMap{
    "upper":      strings.ToUpper,
    "formatDate": func(t time.Time) string {
        return t.Format("2006-01-02")
    },
    "safeHTML": func(s string) template.HTML {
        return template.HTML(s)  // 慎重使用！
    },
}

tmpl := template.Must(
    template.New("example").
        Funcs(funcMap).
        Parse(`{{ .User | upper }} - {{ .Time | formatDate }}`),
)
```

自定义函数应只做**展示层的纯格式化或轻量映射**（日期格式、单位转换、字符串处理）。复杂分支、聚合和权限决策应放在 Go 代码中，不要塞入模板。

## 四、嵌套模板与布局

### 4.1 define / template / block

`{{define "name"}}` 定义命名模板，`{{template "name" .}}` 调用它并传入数据。

```go
const layout = `
{{define "base"}}
<!DOCTYPE html>
<html>
<head><title>{{.Title}}</title></head>
<body>{{template "content" .}}</body>
</html>
{{end}}`

const page = `
{{define "content"}}
<h1>{{.Heading}}</h1>
<p>{{.Body}}</p>
{{end}}`
```

**加载顺序很重要**：Go 模板引擎按加载顺序查找模板，如果多个文件定义了同名模板，**只有第一个被使用**。因此不同页面的内容模板应使用不同的名称。

### 4.2 block 用于可覆盖的默认实现

`{{block "name" .}}...{{end}}` 等价于“定义默认模板 + 调用它”。子模板可以通过 `{{define "name"}}` 覆盖默认实现。

```go
// 主模板
const master = `Names:{{block "list" .}}{{range .}}{{println "-" .}}{{end}}{{end}}`

// 子模板覆盖
const overlay = `{{define "list"}}{{join . ", "}}{{end}}`
```

调用 `masterTmpl.Execute` 输出默认列表；克隆后解析 overlay 再执行，则输出用逗号连接的结果。

### 4.3 Gin 框架中的嵌套模板

在 Gin 中，通常用 `LoadHTMLGlob` 加载所有模板，配合 `{{define}}` 组织布局：

```go
r := gin.Default()
r.LoadHTMLGlob("templates/**/*")

// templates/layout/base.html
// {{define "base"}}
// <html><head><title>{{.title}}</title></head>
// <body>{{template .content .}}</body></html>
// {{end}}

// templates/pages/home.html
// {{define "home-content"}}<p>首页内容</p>{{end}}

// 处理器中
c.HTML(http.StatusOK, "base", gin.H{
    "title":   "Home",
    "content": "home-content",  // 动态指定要渲染的子模板名
})
```

如果子模板名称是动态传入的变量，标准的 `{{template .content .}}` 语法不支持变量作为模板名，需要通过自定义函数（如示例中的 `DynamicTemplate`）来间接实现。

## 五、HTML 安全：html/template 的自动转义

`html/template` 与 `text/template` 共享 API，但增加了**上下文感知的自动转义**。这是 Go 模板最重要的安全特性。

### 5.1 自动转义演示

```go
// text/template：直接输出，产生 XSS
t, _ := template.New("foo").Parse(`Hello, {{.}}!`)
t.Execute(out, "<script>alert('xss')</script>")
// 输出：Hello, <script>alert('xss')</script>!

// html/template：自动转义，安全
t, _ := htmltemplate.New("foo").Parse(`Hello, {{.}}!`)
t.Execute(out, "<script>alert('xss')</script>")
// 输出：Hello, &lt;script&gt;alert('xss')&lt;/script&gt;!
```

转义是**上下文感知**的：同一个 `{{.}}` 出现在 HTML 文本、属性、JavaScript、CSS、URL 中时，会应用不同的转义规则。例如：

```html
<a href="/search?q={{.}}">{{.}}</a>
<!-- 模板引擎在解析时自动改写为： -->
<a href="/search?q={{. | urlescaper | attrescaper}}">{{. | htmlescaper}}</a>
```

### 5.2 安全模型

`html/template` 的安全模型假设**模板作者是可信的，而 Execute 传入的数据不可信**。不应在模板中直接拼接用户输入。如果确实需要输出可信的 HTML 片段，使用 `template.HTML` 类型显式标记，但这**绕过了自动转义**，必须确保内容来源完全可控，绝不可用于用户输入。

## 六、性能优化

### 6.1 核心原则：启动时解析，请求时只执行

`Parse` 包含 I/O 和编译成本，**禁止在每个 HTTP 请求中调用 Parse**。应在 `init()` 或程序启动阶段解析一次，请求路径只做 `Execute`。

```go
// ✅ 正确：包级别初始化
var reportTmpl = template.Must(
    template.ParseFiles("templates/report.html"),
)

func handler(w http.ResponseWriter, r *http.Request) {
    reportTmpl.Execute(w, data)  // 只执行，不解析
}
```

### 6.2 并发安全

模板一旦解析完成，`Execute` 可以安全地并发调用。但如果多个 goroutine 共享同一个 `io.Writer`，输出可能交错。每个请求应使用独立的 `Writer`（如 `http.ResponseWriter`）。

### 6.3 需要不同数据隔离时使用 Clone

如果需要在同一套模板基础上为不同请求做数据隔离的变体，使用 `Clone()` 创建副本。

### 6.4 避免不必要的转义开销

`html/template` 的自动转义会带来一定的性能开销。如果输出的是纯文本（如生成代码、配置文件），使用 `text/template` 可以避免转义成本。只有在生成 HTML 时才应使用 `html/template`。

## 七、测试策略

模板测试的核心是**构造数据 → 执行 → 断言输出**。推荐的模式是表驱动测试。

```go
func TestLetterTemplate(t *testing.T) {
    const letter = `Dear {{.Name}},{{if .Attended}} 感谢您的到来。{{else}} 很遗憾您未能出席。{{end}}`

    tests := []struct {
        name     string
        data     Recipient
        contains string
    }{
        {
            name:     "attended",
            data:     Recipient{Name: "Mildred", Attended: true},
            contains: "感谢您的到来",
        },
        {
            name:     "not attended",
            data:     Recipient{Name: "John", Attended: false},
            contains: "很遗憾您未能出席",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tmpl := template.Must(template.New("letter").Parse(letter))
            var buf bytes.Buffer
            if err := tmpl.Execute(&buf, tt.data); err != nil {
                t.Fatal(err)
            }
            if !strings.Contains(buf.String(), tt.contains) {
                t.Errorf("output %q does not contain %q", buf.String(), tt.contains)
            }
        })
    }
}
```

对于 HTML 模板，还应测试**特殊字符是否被正确转义**，以及子模板的嵌套执行是否正确。

## 八、实战：完整 HTTP 服务示例

```go
package main

import (
    "html/template"
    "log"
    "net/http"
    "time"
)

type User struct {
    Name      string
    Age       int
    JoinedAt  time.Time
    IsPremium bool
}

var tmpl = template.Must(
    template.New("user").
        Funcs(template.FuncMap{
            "formatDate": func(t time.Time) string {
                return t.Format("2006-01-02")
            },
        }).
        Parse(`
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"><title>用户信息</title></head>
<body>
<h1>{{.Name}}</h1>
<p>年龄：{{.Age}}</p>
<p>注册日期：{{.JoinedAt | formatDate}}</p>
{{if .IsPremium}}
    <span class="badge">尊贵会员</span>
{{else}}
    <span>普通用户</span>
{{end}}
</body>
</html>`),
)

func userHandler(w http.ResponseWriter, r *http.Request) {
    user := User{
        Name:      "张三",
        Age:       28,
        JoinedAt:  time.Now(),
        IsPremium: true,
    }
    if err := tmpl.Execute(w, user); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}

func main() {
    http.HandleFunc("/", userHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

模板在包级别解析一次，请求路径只执行 `Execute`；自定义 `formatDate` 函数在 `Parse` 之前通过 `Funcs` 注册；`html/template` 自动对 `{{.Name}}` 等内容进行 HTML 转义。

## 九、最佳实践总结

**解析时机**：模板在启动时解析一次，绝不在请求路径中重复解析。使用 `template.Must` 在初始化阶段暴露解析错误。

**函数注册顺序**：`Funcs()` 必须在 `Parse()` 之前调用，否则解析阶段就会失败。

**选择正确的包**：生成 HTML 用 `html/template`；生成代码、配置、纯文本用 `text/template`。绝不能用 `text/template` 渲染用户可见的 HTML 页面。

**安全边界**：将 `template.HTML`、`template.JS`、`template.CSS` 视为“危险类型”。它们绕过自动转义，只应用于来源完全可控的内容，严禁包裹用户输入。

**逻辑归属**：模板只负责展示。复杂条件分支、数据聚合、权限判断放在 Go 代码中完成，模板接收的是“已经准备好的展示数据”。

**模板命名**：嵌套模板中不同页面的内容块使用**唯一名称**，避免加载顺序导致同名模板被覆盖的隐蔽 bug。

**空白控制**：生成结构化文本（代码、JSON、YAML）时，在 `range`、`if`、`end` 动作上使用 `-` 修剪符精确控制输出格式，避免多余换行破坏结构。

**并发写入**：多 goroutine 并发执行同一模板时，为每个执行者提供独立的 `io.Writer`，避免输出交错。