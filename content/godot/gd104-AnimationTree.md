---
title: 'Godot AnimationTree'
date: '2026-11-04T09:56:14+08:00'
categories: [godot,animation]
tags: [AnimationTree]
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


`AnimationTree` 是 Godot 动画系统的**决策层**。它本身不包含任何动画数据，而是指向一个已有的 `AnimationPlayer`，在运行时根据你的逻辑（状态切换、输入方向、速度值）动态决定“现在该播放什么、如何混合”。当你发现需要在多条动画之间**平滑过渡**、**按方向混合**，或在基础动画上**叠加一次性动作**时，就该引入 `AnimationTree` 了。

---

### 🧭 三种核心组织方式：先选对“大脑”

`AnimationTree` 的根节点决定了它的调度方式，最常用的是以下三种：

| 根节点类型 | 解决的问题 | 典型场景 |
| :--- | :--- | :--- |
| **`AnimationNodeStateMachine`** | 离散状态的切换与过渡 | `idle` ↔ `walk` ↔ `jump` ↔ `attack` |
| **`AnimationNodeBlendSpace2D`** | 连续方向/速度的混合 | 8 方向移动、`walk`→`run` 的速度混合 |
| **`AnimationNodeBlendTree`** | 分层与叠加（配合 `OneShot` 等节点） | 走路时开枪、受击闪烁叠加在基础动画上 |

实际项目中，**状态机与混合空间通常是嵌套配合的**：状态机负责“当前处于什么大状态”（如 `Locomotion`），而该状态内部的具体方向或速度变化，交给嵌套的 `BlendSpace2D` 处理。

---

### ⚙️ 前置设置：两步让树跑起来

在写任何代码之前，`AnimationTree` 需要两个明确的配置，否则它不会驱动任何东西：

1. **`anim_player` 指向数据源**：在检查器中将 `AnimationTree` 的 `anim_player` 属性指向场景中已有的 `AnimationPlayer`。
2. **`active = true`**：`AnimationTree` 默认不激活，必须在检查器中勾选，或在 `_ready()` 中设置 `active = true`。

---

### 🧩 示例一：最简状态机 —— `travel()` 事件驱动

**适用场景**：由代码逻辑直接命令的状态切换，如跳跃、攻击、死亡。

**树的结构**（在 `AnimationTree` 编辑器中创建 `AnimationNodeStateMachine`，添加状态并连线）：

```text
StateMachine
├── Idle
├── Walk
└── Jump
```

**关键代码**：状态机播放控制器通过 `playback` 参数获取，`travel()` 会**立即中断**当前状态并转向目标状态，并自动寻找最短路径。

```gdscript
extends CharacterBody2D

@onready var anim_tree: AnimationTree = $AnimationTree
@onready var state_machine: AnimationNodeStateMachinePlayback = anim_tree.get("parameters/StateMachine/playback")

func _physics_process(_delta: float) -> void:
    var input_dir := Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    var target_state: String

    if Input.is_action_just_pressed("jump"):
        state_machine.travel("Jump")
        return  # 这一帧不再处理移动状态，让跳跃优先

    if input_dir != Vector2.ZERO:
        target_state = "Walk"
    else:
        target_state = "Idle"

    # 避免每帧重复 travel 同一状态（travel 到当前状态通常不会重置）
    if state_machine.get_current_node() != target_state:
        state_machine.travel(target_state)
```

**要点**：`travel()` 适合**事件**（一次性、需要立即打断当前动作的情况）；对于 `idle` ↔ `walk` 这类**连续状态**，下一节的“条件驱动”是更稳健的选择。

---

### 🧩 示例二：条件驱动状态切换 —— 声明式过渡

**适用场景**：`idle` ↔ `walk` 这类由持续状态（速度是否为零）决定的切换，不希望用命令式的 `travel()` 每帧去“推”。

**树的结构**：在状态机的过渡线（Transition）上，设置 **Advance Condition**，填入一个布尔参数名：

```text
Idle → Walk 的过渡线：Advance Condition = "moving"
Walk → Idle 的过渡线：Advance Condition = "not_moving"
```

**关键代码**：每一帧写入条件参数，条件满足时状态机**自动**完成过渡。

```gdscript
extends CharacterBody2D

@onready var anim_tree: AnimationTree = $AnimationTree

func _physics_process(_delta: float) -> void:
    var input_dir := Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    var is_moving := input_dir.x != 0.0

    # 两个条件互斥，且每帧都写入，避免“残留 true”把状态机卡死
    anim_tree.set("parameters/moving", is_moving)
    anim_tree.set("parameters/not_moving", not is_moving)
```

**两条铁律**：每个条件**每帧都要写入**（残留的 `true` 会让状态机永远卡在一个状态）；条件之间应当**互斥**，否则过渡结果会取决于节点在编辑器中的顺序，行为不可预测。

---

### 🧩 示例三：`BlendSpace2D` 方向混合 —— 8 方向移动

**适用场景**：角色需要根据输入方向，在多个方向的动画之间**连续插值**，而不是硬切 8 个独立状态。

**树的结构**：在状态机内创建一个 `Locomotion` 状态，将该状态的内容设为 `BlendSpace2D`，并在 2D 平面上放置各方向的动画点：

| 动画 | Blend 坐标 (x, y) |
| :--- | :--- |
| `walk_down` | `(0, 0.65)` |
| `walk_up` | `(0, -0.65)` |
| `walk_left` | `(-0.65, 0)` |
| `walk_right` | `(0.65, 0)` |
| `walk_down_right` | `(0.45, 0.45)` |

**关键代码**：每一帧将输入方向写入 `blend_position`；**停止移动时也要写入 `Vector2.ZERO`**，否则角色会“停在最后一次的方向”上，看起来像 idle 动画出了 bug。

```gdscript
extends CharacterBody2D

@onready var anim_tree: AnimationTree = $AnimationTree
@onready var state_machine: AnimationNodeStateMachinePlayback = anim_tree.get("parameters/playback")

const SPEED := 200.0

func _physics_process(_delta: float) -> void:
    var input_dir := Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")

    # 方向混合：每一帧都写入，停止时写 Vector2.ZERO
    anim_tree.set("parameters/Locomotion/blend_position", input_dir.normalized())

    # 条件驱动 Idle ↔ Locomotion（假设过渡条件已设好）
    anim_tree.set("parameters/moving", input_dir != Vector2.ZERO)
    anim_tree.set("parameters/not_moving", input_dir == Vector2.ZERO)

    velocity = input_dir * SPEED
    move_and_slide()
```

`BlendSpace2D` 的坐标值**不需要是单位长度**，节点在混合时会自动归一化，但代码写入的坐标系必须与编辑器中放置动画点的坐标系一致。

---

### 🧩 示例四：`OneShot` 一次性叠加 —— 走路时开枪

**适用场景**：在基础动画（走/跑/待机）之上，**短暂播放一个上半身或全身动作**，播完后自动返回基础动画，不打断底层状态。

**树的结构**：将根节点改为 `AnimationNodeBlendTree`，放入一个 `OneShot` 节点，将基础动画接到 `OneShot` 的 `in` 端口，一次性动作（如 `shoot`）接到 `shot` 端口。

**关键代码**：通过 `request` 参数触发一次性动画；`OneShot` 会在播完后**自动清除请求**并回到 `in` 端口的基础动画。

```gdscript
extends CharacterBody2D

@onready var anim_tree: AnimationTree = $AnimationTree

func _process(_delta: float) -> void:
    if Input.is_action_just_pressed("shoot"):
        # 触发一次性射击动画，播完后自动回到走路/待机
        anim_tree.set("parameters/OneShot/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE)

    if Input.is_action_just_pressed("melee"):
        # 另一种一次性动作，同样叠加在基础动画上
        anim_tree.set("parameters/MeleeOneShot/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE)
```

如果需要在一次性动画**尚未播完时强制打断**，可以使用 `ONE_SHOT_REQUEST_ABORT`（硬切）或 `ONE_SHOT_REQUEST_FADE_OUT`（淡出中断）。

---

### 📋 参数路径速查

`AnimationTree` 的所有控制都通过**字符串路径**读写参数，路径由编辑器中的节点名称决定：

| 操作 | 路径写法 |
| :--- | :--- |
| 获取状态机播放控制器 | `"parameters/StateMachine/playback"`（或 `"parameters/playback"`，视层级而定） |
| 触发状态机 `travel()` | `state_machine.travel("StateName")` |
| 读写条件参数 | `"parameters/moving"` |
| 设置 BlendSpace2D 位置 | `"parameters/Locomotion/blend_position"` |
| 触发 OneShot | `"parameters/OneShot/request"` |

**路径不匹配是 `AnimationTree` 最常见的“不工作”原因**：检查器中的参数名必须与代码中的字符串**完全一致**（包括大小写和层级），否则设置会被静默忽略。

---

### ⚠️ 三个高频陷阱

**① 条件残留**：Advance Condition 对应的布尔参数如果某一帧没有被写入，它会保留上一帧的值。一个残留的 `true` 会让状态机永久卡死在某条过渡上。**每个条件每帧都写入**是唯一可靠的做法。

**② `travel()` 与条件混用时的竞争**：`travel()` 是“排队”执行的，而条件过渡在下一帧就会生效。如果先用 `travel("Jump")`，但下一帧条件驱动又把状态拉回 `Locomotion`，跳跃动画可能根本来不及播放。**事件用 `travel()`，持续状态用条件**，不要用条件去抢 `travel()` 刚设置的目标。

**③ 忘记设置 `active = true`**：`AnimationTree` 创建后默认不激活，即使 `anim_player` 已正确指向，它也不会输出任何动画。

---

### 💎 学习路径小结

从 **`travel()` 驱动的事件状态机**开始，把 `idle → walk → jump` 跑通；然后引入 **Advance Condition** 替代 `travel()` 来管理 `idle ↔ walk`；再在一个 `Locomotion` 状态内部嵌入 **`BlendSpace2D`** 处理方向；最后用 **`OneShot`** 解决“在移动中叠加动作”的需求。每一步都只增加一个概念，而不是一次性把所有节点类型堆上去——`AnimationTree` 的复杂度来自**参数路径的精确性**和**条件逻辑的严谨性**，而不是节点本身的数量。