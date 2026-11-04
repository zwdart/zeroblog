---
title: 'Godot Tween'
date: '2026-11-04T09:51:01+08:00'
categories: [godot,animation]
tags: [tween]
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


`Tween` 是 Godot 中用于“在代码里顺手做一小段动画”的轻量级工具。它的核心用途很单纯：**将一个属性从当前值平滑地插值到目标值**，创建后自动运行，完成后自动释放，非常适合 UI 反馈、简单的位移/透明度变化、镜头震动这类“即用即弃”的动效。

---

### 🧱 先建立正确的心智模型

使用 `Tween` 时，有几个前置规则需要先刻在脑子里：

**① 必须用 `create_tween()` 创建**

`Tween.new()` 是无效的，不能用来做补间。正确的入口只有两个：

```gdscript
# 方式一：从节点创建，Tween 会绑定到该节点的生命周期
var tween = create_tween()

# 方式二：从场景树创建，不绑定特定节点
var tween = get_tree().create_tween()
```

**② Tween 本质上是一条“时间线”**

你往这条时间线上添加的每一个 `tween_property()`、`tween_callback()`、`tween_interval()`，都是一个 **Tweener**。默认情况下，**这些 Tweener 会按添加顺序一个接一个地串行执行**。

**③ 默认是串行，`set_parallel(true)` 可以改为并行**

这是控制 Tween 行为最关键的一个开关，后面会用具体示例展开。

---

### ✂️ 核心 API：你只需要先掌握这几个

| 方法 | 作用 |
| :--- | :--- |
| **`tween_property(object, property, final_val, duration)`** | 最核心的方法，在 `duration` 秒内将 `property` 插值到 `final_val` |
| **`tween_callback(callable)`** | 在时间线的某个点执行一个函数 |
| **`tween_interval(time)`** | 在时间线上插入一段纯等待 |
| **`set_parallel(true)`** | 让后续的 Tweener 并行执行，而不是串行 |
| **`chain()`** | 从并行模式切回串行模式 |
| **`set_loops(n)`** | 设置循环次数，`0` 或不传参表示无限循环 |
| **`set_ease()` / `set_trans()`** | 控制缓动曲线（加速/减速的节奏） |
| **`.from(value)`** | 指定起始值，不依赖属性的当前值 |
| **`.as_relative()`** | 将目标值视为相对偏移量 |

---

### 🍳 常用可运行示例

以下示例均假设脚本挂在某个节点上，可以直接复制到 `_ready()` 或按钮回调中运行。

---

#### 示例 1：基础位移 —— 让节点滑到指定位置

```gdscript
func _ready() -> void:
    var tween = create_tween()
    tween.tween_property(self, "position", Vector2(400, 300), 1.0)
        .set_trans(Tween.TRANS_CUBIC)
        .set_ease(Tween.EASE_OUT)
```

`set_trans(Tween.TRANS_CUBIC)` 让运动带有“起步快、收尾缓”的节奏感，`.set_ease(Tween.EASE_OUT)` 表示缓出。常用的组合是 **`TRANS_CUBIC` + `EASE_OUT`**，对大多数 UI 动效都很自然。

---

#### 示例 2：淡入 / 淡出（UI 最常用）

```gdscript
func fade_in(node: CanvasItem, duration := 0.3) -> void:
    node.modulate.a = 0.0
    create_tween().tween_property(node, "modulate:a", 1.0, duration)

func fade_out(node: CanvasItem, duration := 0.3) -> void:
    create_tween().tween_property(node, "modulate:a", 0.0, duration)
```

属性路径用 `"modulate:a"` 直接绑定到透明度分量，比操作整个 `Color` 更清晰。

---

#### 示例 3：串行序列 —— 变红 → 缩小 → 释放

这是官方文档中的标准示例，展示了 Tween 的“时间线”本质：

```gdscript
func _ready() -> void:
    var tween = create_tween()
    tween.tween_property($Sprite, "modulate", Color.RED, 1.0)
    tween.tween_property($Sprite, "scale", Vector2.ZERO, 1.0)
    tween.tween_callback($Sprite.queue_free)
```

三个步骤严格按顺序执行：先变红，再缩小到零，最后调用 `queue_free()` 释放节点。

---

#### 示例 4：延迟回调 —— 两秒后执行某个函数

```gdscript
func _ready() -> void:
    var tween = create_tween()
    tween.tween_callback(_on_delayed).set_delay(2.0)

func _on_delayed() -> void:
    print("2 秒后执行")
```

`.set_delay(2.0)` 让回调在加入时间线后等待 2 秒再执行，很适合替代手动的计时器来做简单的延迟逻辑。

---

#### 示例 5：并行动画 —— 面板滑入的同时淡入

```gdscript
func show_panel(panel: Control) -> void:
    panel.modulate.a = 0.0
    panel.position.x = -200

    var tween = create_tween().set_parallel(true)
    tween.tween_property(panel, "position:x", 100.0, 0.4)
        .set_trans(Tween.TRANS_CUBIC).set_ease(Tween.EASE_OUT)
    tween.tween_property(panel, "modulate:a", 1.0, 0.4)
```

调用 `set_parallel(true)` 之后，**后续添加的所有 Tweener 会同时开始**，而不是排队执行。这是 UI 弹出动画最常用的模式：位移、透明度、缩放同时进行，整体感觉会更利落。

---

#### 示例 6：并行结束后接回调 —— `chain()` 的用法

如果并行块结束后还需要执行一段串行逻辑，必须用 `chain()` 切回串行：

```gdscript
func popup_animation(node: Control) -> void:
    var tween = create_tween().set_parallel(true)
    tween.tween_property(node, "scale", Vector2(1.1, 1.1), 0.2)
    tween.tween_property(node, "modulate:a", 1.0, 0.2)
    tween.chain().tween_callback(_on_popup_ready)
```

`chain()` 的作用就是把“并行模式”关掉，恢复为串行，从而让后面的 `tween_callback` 等到前面的并行动画全部完成后才触发。

---

#### 示例 7：无限循环的往返移动

```gdscript
func start_bobbing() -> void:
    var start_x := position.x
    var end_x := position.x + 100.0

    var tween = create_tween().set_loops() # 无限循环
    tween.tween_property(self, "position:x", end_x, 1.0).from(start_x)
    tween.tween_property(self, "position:x", start_x, 1.0).from(end_x)
```

`set_loops()` 不传参数表示无限循环。`.from()` 明确指定每段的起点，避免依赖属性当前值，让往返逻辑更可控。

---

#### 示例 8：相对位移 —— `as_relative()` 的用法

```gdscript
func shake() -> void:
    var tween = create_tween()
    tween.tween_property(self, "position:x", 10.0, 0.05).as_relative()
    tween.tween_property(self, "position:x", -20.0, 0.05).as_relative()
    tween.tween_property(self, "position:x", 10.0, 0.05).as_relative()
```

`.as_relative()` 让目标值变成**相对当前值的偏移量**，非常适合做不需要关心绝对坐标的震动、抖动效果。

---

#### 示例 9：弹性缩放 —— 按钮按压反馈

```gdscript
func press_feedback(button: Control) -> void:
    var tween = create_tween()
    tween.tween_property(button, "scale", Vector2(0.9, 0.9), 0.1)
    tween.tween_property(button, "scale", Vector2.ONE, 0.3)
        .set_trans(Tween.TRANS_ELASTIC)
        .set_ease(Tween.EASE_OUT)
```

先用很短的时间缩小，再用弹性缓动弹回原始大小，是按钮点击反馈中很常见的“Q 弹”手感。

---

#### 示例 10：控制生命周期 —— 保存引用，随时终止

Tween 创建后会自动运行，但如果你需要中途取消（比如玩家在动画途中离开了状态），应当保存引用并调用 `kill()`：

```gdscript
var _move_tween: Tween

func move_to(target: Vector2) -> void:
    if _move_tween and _move_tween.is_valid():
        _move_tween.kill()  # 先终止旧 Tween，避免多 Tween 争抢同一属性
    _move_tween = create_tween()
    _move_tween.tween_property(self, "position", target, 0.5)

func cancel_movement() -> void:
    if _move_tween and _move_tween.is_valid():
        _move_tween.kill()
```

一个重要的实践规则：**不要用多个 Tween 同时驱动同一个属性**，否则行为会变得不可预测。在启动新 Tween 之前，先 `kill()` 旧的引用。

---

### 🎚️ 缓动速查：`set_trans` 与 `set_ease` 怎么选

| 场景 | 推荐组合 |
| :--- | :--- |
| UI 滑入 / 淡入 | `TRANS_CUBIC` + `EASE_OUT` |
| 弹簧/弹性反馈 | `TRANS_ELASTIC` + `EASE_OUT` |
| 自然往返（呼吸感） | `TRANS_SINE` + `EASE_IN_OUT` |
| 快速起步、缓慢收尾 | `TRANS_QUAD` + `EASE_OUT` |
| 强调“砸下来”的感觉 | `TRANS_BACK` + `EASE_IN` |

`set_ease` 和 `set_trans` 可以链式调用在单个 `tween_property` 上，也可以设置在 `create_tween()` 后作为整个 Tween 的默认值。

---

### 💎 总结

`Tween` 的学习曲线很短，关键是把三个概念分清楚：**时间线是串行的**（用 `set_parallel` 切换并行）、**Tweener 是可链式配置的**（`.from()`、`.set_trans()`、`.set_delay()` 都作用在单个步骤上）、**Tween 对象本身是可终止的**（保存引用，必要时 `kill()`）。把上面十个示例跑一遍，再组合出“并行位移+淡入，结束后回调”这类复合动效，你就能覆盖大多数日常需求了。