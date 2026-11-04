---
title: 'Godot AnimationPlayer'
date: '2026-11-04T09:53:13+08:00'
categories: [godot,animation]
tags: [AnimationPlayer]
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


`AnimationPlayer` 是 Godot 动画系统的**数据核心**。与“即用即弃”的 `Tween` 不同，它把动画作为**资源**保存在节点里，通过可视化的时间轴进行编排，可以在编辑器中精细调整节奏，再用代码决定何时播放哪一段。如果你的动画涉及多个属性、需要精确到帧的时序，或者包含音效、函数调用等非属性事件，`AnimationPlayer` 是比 `Tween` 更合适的选择。

---

### 🧱 先建立正确的场景结构

`AnimationPlayer` 继承自**裸 `Node`**，而不是 `Node2D` 或 `Node3D`。这意味着它**不传递变换**——如果把带有 2D/3D 变换的节点设为它的子节点，子节点不会继承父级的位移、旋转或缩放，视觉上会出现位置偏移或异常。

正确的结构是让 `AnimationPlayer` 与被动画的节点**平级**：

```text
Player (CharacterBody2D)
├── Sprite2D
└── AnimationPlayer
```

`AnimationPlayer` 可以通过**轨道路径**驱动任意兄弟或子节点的属性：位置、旋转、缩放、`modulate`、`visible`、碰撞形状的 `disabled` 状态，甚至方法调用和音频播放。

---

### 🗂️ 轨道类型：不止是属性

在动画面板中点击“添加轨道”，你会看到多种类型。常用的是以下四种：

| 轨道类型 | 作用 | 典型用途 |
| :--- | :--- | :--- |
| **属性轨道** (Property / Value) | 插值任意属性 | 位置、旋转、透明度、着色器参数 |
| **方法轨道** (Call Method) | 在精确时间点调用函数 | 生成伤害判定、播放音效、切换状态 |
| **音频轨道** (Audio Playback) | 将音频锁定在时间轴上 | 脚步声与落脚点精确同步 |
| **贝塞尔轨道** (Bezier) | 用曲线精细控制数值节奏 | 非线性的 UI 动效、特殊缓动 |

**方法轨道的关键配置**：默认的调用模式是 `CALL_MODE_CONTINUOUS`，在关键帧前后拖拽播放头时可能**每帧重复触发**。对于“伤害判定只该发生一次”的逻辑，必须在代码中将调用模式设为 `CALL_MODE_DISCRETE`，确保函数在越过关键帧时**只执行一次**。

---

### ▶️ 代码控制：播放、混合与队列

#### 基础播放与状态驱动

```gdscript
@onready var anim_player: AnimationPlayer = $AnimationPlayer

func _physics_process(_delta: float) -> void:
    var input_dir := Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    if input_dir != Vector2.ZERO:
        anim_player.play("walk")
    else:
        anim_player.play("idle")
```

一个重要的细节：**对正在播放的同一动画重复调用 `play()` 不会重新开始**，也不会报错。因此可以在 `_physics_process` 中放心地每帧调用 `play()`，根据状态切换动画。

#### 常用控制方法速查

```gdscript
anim_player.play("attack")                # 正向播放
anim_player.play_backwards("attack")      # 反向播放
anim_player.queue("idle")                 # 当前动画结束后自动播放下一个
anim_player.stop()                        # 立即停止
anim_player.pause()                       # 暂停在当前帧
anim_player.play()                        # 从暂停位置恢复
anim_player.speed_scale = 2.0             # 加速播放
anim_player.seek(0.5)                     # 跳转到 0.5 秒处
```

`queue()` 是串联动画的轻量方式：先播放 `attack`，`attack` 播完后自动进入 `idle`，不需要手动等待信号。

#### 平滑过渡：利用混合时间

`play()` 的第二个参数控制混合时间，让两段动画之间**交叉淡入淡出**，而不是硬切换：

```gdscript
# 在 0.2 秒内从当前动画混合过渡到 run
anim_player.play("run", 0.2)
```

如果不传这个参数，Godot 会使用动画面板中“编辑过渡”（Edit Transitions）里为每对动画预设的混合时间。对于 `idle ↔ walk` 这类频繁切换的状态，设置 **0.1–0.2 秒**的混合时间能显著提升手感。

---

### 🎬 方法轨道：把逻辑事件焊在时间线上

属性轨道管“视觉”，方法轨道管“事件”。例如在攻击动画的第 0.3 秒生成伤害判定：

在编辑器中添加一条**方法轨道**，路径指向挂有目标脚本的节点，在第 0.3 秒插入关键帧并填入方法名 `spawn_hitbox`。然后在代码中确保调用模式为单次触发：

```gdscript
# 在 _ready() 或动画准备阶段，将方法轨道设为 DISCRETE
func _ready() -> void:
    var anim := anim_player.get_animation("attack")
    for i in anim.get_track_count():
        if anim.track_get_type(i) == Animation.TYPE_METHOD:
            anim.track_set_call_mode(i, Animation.CALL_MODE_DISCRETE)
            break
```

被调用的函数必须存在于目标节点上：

```gdscript
func spawn_hitbox() -> void:
    # 生成伤害区域、播放特效等，只会执行一次
    var hitbox := preload("res://hitbox.tscn").instantiate()
    add_child(hitbox)
```

如果需要在播放中途**动态改变调用的方法或参数**，方法轨道的关键帧值本身是一个字典 `{ "method": "函数名", "args": [...] }`，可以在运行时通过 `track_insert_key` 替换。

---

### 📚 动画库：组织多个动画

一个 `AnimationPlayer` 可以保存多条动画。出于复用性考虑，动画被注册在 **`AnimationLibrary`** 资源中。如果你直接新建动画而不指定库，它会进入默认的 `[Global]` 库，直接 `play("idle")` 即可。

当动画数量增多时，按用途分库会更清晰：

```gdscript
# 运行时加载额外的动画库
var combat_lib: AnimationLibrary = load("res://animations/combat_lib.tres")
anim_player.add_animation_library("combat", combat_lib)

# 播放指定库中的动画
anim_player.play("combat/slash")
```

分库的好处是**按需加载**：基础移动动画可以常驻，战斗动画库只在进入战斗时加载，减少内存占用。

---

### ⚠️ 两个必须避开的坑

**① 不要给根节点直接添加动画**

动画中的 `position` 是**相对于父节点**的坐标。如果直接给场景根节点打位置关键帧，当这个场景被实例化到另一个场景中时，动画会以**新的父节点位置**作为原点，导致节点“跳”到错误的位置。解决办法是给场景加一个空白根节点，把原来的内容包进去，动画只作用于内部的子节点。

**② 移动或重命名节点后，轨道可能失效**

`AnimationPlayer` 的轨道路径是**节点路径字符串**。如果你在创建动画后移动或重命名了被动画的节点，轨道路径不会自动更新，播放时会出现“Node not found”错误，属性不再被驱动。双击轨道可以手动编辑路径，或者在调整场景结构后重新绑定轨道。

---

### 🧩 完整可运行示例：带过渡的移动状态机

将以下脚本挂在一个 `CharacterBody2D` 上，确保场景中有一个 `AnimationPlayer` 包含 `idle` 和 `walk` 两条动画：

```gdscript
extends CharacterBody2D

@onready var anim_player: AnimationPlayer = $AnimationPlayer

const SPEED := 200.0
const BLEND_TIME := 0.15

func _physics_process(_delta: float) -> void:
    var input_dir := Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")

    if input_dir != Vector2.ZERO:
        velocity = input_dir * SPEED
        # 如果当前不在 walk，用 0.15 秒混合进入
        if anim_player.current_animation != "walk":
            anim_player.play("walk", BLEND_TIME)
    else:
        velocity = Vector2.ZERO
        if anim_player.current_animation != "idle":
            anim_player.play("idle", BLEND_TIME)

    move_and_slide()
```

这个示例的关键在于**用 `current_animation` 判断是否需要切换**，避免每帧重复调用 `play()` 导致混合反复重置。`BLEND_TIME` 让 `idle` 和 `walk` 之间平滑交叉淡化，而不是生硬跳变。

`AnimationPlayer` 的强项是**时间轴编排**——一旦你把关键帧、方法轨道和混合时间配置好，代码端只需要关心“何时切换到哪条动画”，决策逻辑可以保持得非常薄。当你需要状态机或方向混合空间时，`AnimationTree` 才是下一步的升级，而 `AnimationPlayer` 始终是它背后的数据底座。