---
title: 'Godot Animation介绍'
date: '2026-11-04T09:44:43+08:00'
categories: [godot,animation]
tags: [animation]
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



Godot 的动画系统以**简洁但极具深度**著称。理解它的关键在于建立一个清晰的层次观：**`Tween`、`AnimationPlayer` 和 `AnimationTree` 三者并非替代关系，而是分别针对不同场景、逐级递进的工具**。

---

### 🗺️ 先看全景：你该在什么时候用什么？

社区里一条非常实用的“黄金法则”是：**从 `AnimationPlayer` 开始；当你需要动画之间的混合、过渡或分层时，再引入 `AnimationTree`**。

| 工具 | 最适合的场景 | 关键特征 |
| :--- | :--- | :--- |
| **`Tween`** | 代码驱动的一次性短动画：按钮悬停、屏幕震动、UI 淡入、伤害闪烁 | 纯代码创建，运行一次后自动释放 |
| **`AnimationPlayer`** | 需要在编辑器中“编排”的动画：精灵帧序列、复杂运动、过场动画、UI 过渡 | 可视化时间轴，可复用，能驱动几乎任何属性 |
| **`AnimationTree`** | 需要**状态管理**或**混合**的角色动画：`idle→walk→run`、8 方向移动、上下半身分层 | 包裹 `AnimationPlayer`，用状态机或混合树来调度播放 |

你的学习路径可以按照 **`Tween` → `AnimationPlayer` → `AnimationTree`** 的顺序推进，下面逐层展开。

---

### 🌱 第一层：`Tween` —— 代码里的“一次性动效”

`Tween` 是三者中最轻量的。它不活在场景树里，而是由代码通过 `create_tween()` 创建，专注于将某个属性从当前值平滑插值到目标值。

```gdscript
# 让节点在 1 秒内移动到 (500, 300)
var tween = create_tween()
tween.tween_property(self, "position", Vector2(500, 300), 1.0)
    .set_ease(Tween.EASE_OUT)
    .set_trans(Tween.TRANS_CUBIC)
```

**核心心智模型**：`Tween` 是“即用即走”的。如果只是一个按钮弹一下、一个面板滑进来，用它最直接，不需要在场景里挂节点，也不需要打开动画面板。

---

### ✂️ 第二层：`AnimationPlayer` —— 你的万能动画剪辑台

`AnimationPlayer` 是整个动画系统的**数据基础**。它本身是一个 `Node`，内部可以存放多条**动画片段**，每条动画由若干**轨道**组成，轨道上放置**关键帧**。

#### 关键概念：轨道与关键帧

一条**轨道**绑定到“某个节点的某个属性”（例如 `Sprite2D:position`）。**关键帧**记录该属性在某一时刻的值；两个关键帧之间，引擎会自动插值，让属性平滑变化。

`AnimationPlayer` 远不止能做位移。它可以驱动**检查器里几乎任何可插值的属性**：颜色、透明度、碰撞形状的启用状态、着色器参数等等。

#### 不止是属性：轨道类型全览

除了最常用的属性轨道，`AnimationPlayer` 还支持多种专用轨道，这是它成为“剪辑台”的关键：

| 轨道类型 | 作用 | 典型用途 |
| :--- | :--- | :--- |
| **属性轨道** (Property) | 插值节点属性 | 位置、旋转、缩放、颜色 |
| **方法轨道** (Call Method) | 在精确时间点调用函数 | 在攻击动画第 0.3 秒生成伤害判定、播放音效、触发状态切换 |
| **音频轨道** (Audio Playback) | 将音频锁定在时间轴上播放 | 让脚步声与脚落地精确同步 |
| **贝塞尔曲线轨道** (Bezier) | 用曲线精细控制数值变化节奏 | 非线性的 UI 动效、特殊的缓动需求 |
| **动画轨道** (Animation) | 播放其他动画 | 在一条时间轴上串联多个片段 |

**方法轨道的一个重要细节**：它默认的调用模式是 `CONTINUOUS`，在关键帧前后拖拽播放头时可能会反复触发。对于“伤害判定只该发生一次”的逻辑，应当在代码中将调用模式设为 `CALL_MODE_DISCRETE`，确保函数在越过关键帧时只执行一次。

#### ⚠️ 一个必须避开的坑

不要将带有 2D/3D 变换的节点设为 `AnimationPlayer` 的**直接子节点**。因为 `AnimationPlayer` 继承自裸 `Node`，不传递变换，这会导致子节点的视觉表现异常。正确的结构是让 `AnimationPlayer` 与 `Sprite2D` 等节点**平级**，而不是成为它的父节点。

---

### 🧠 第三层：`AnimationTree` —— 状态机与混合的大脑

当你的动画需求从“播放一条片段”升级为“在多个片段之间智能切换和混合”时，`AnimationTree` 就该登场了。

它的基本结构是：`AnimationTree` 通过 `anim_player` 属性指向一个 `AnimationPlayer`，自身在运行时以**参数**的形式对外暴露控制接口。

#### 两种核心组织方式：状态机 vs 混合空间

**① 状态机 (`AnimationNodeStateMachine`)** —— 管理“离散状态”的切换

适合 `idle`、`walk`、`jump`、`attack` 这类**互斥状态**。你在编辑器中画出状态节点和过渡连线，通过代码驱动状态迁移。

最常用的两种驱动方式：

- **`travel()`**：命令式切换。在 `_physics_process` 中根据游戏逻辑判断目标状态并调用。
  ```gdscript
  var playback = anim_tree.get("parameters/StateMachine/playback")
  playback.travel("Run")
  ```
  

- **Advance Condition（推进条件）**：声明式切换。在过渡线上设置一个布尔条件名，代码中设置该参数，条件满足时自动过渡。适合**受击、死亡**等事件驱动的切换。

此外，**Auto Advance** 可以用于**连招链**：`Attack1 → Attack2` 无需条件，前一段播完自动进入下一段。

**② 混合空间 (`BlendSpace1D` / `BlendSpace2D`)** —— 管理“连续变化”

适合**方向**和**速度**这类连续量。

- **`BlendSpace1D`**：单一参数驱动。例如用速度值在 `walk` 和 `run` 之间连续混合。
  ```gdscript
  anim_tree["parameters/BlendSpace1D/blend_position"] = inverse_lerp(walk_speed, run_speed, velocity.length())
  ```
  

- **`BlendSpace2D`**：二维向量驱动。将 8 个方向的动画放置在平面上的对应坐标，用输入方向向量直接驱动，实现**8 方向移动**的自然混合。

**实际项目中，两者是配合使用的**：状态机负责“当前该处于什么大状态”（比如 `Locomotion` 状态），而该状态内部的 `walk/run` 切换或方向混合则交给混合空间处理。

---

### 🦴 专项：骨骼动画（2D 与 3D）

**2D 骨骼**通过 `Skeleton2D` + `Bone2D` 层级实现。`AnimationPlayer` 对 `Bone2D` 的旋转属性打关键帧，即可驱动骨骼摆动。

如果需要**程序化**的肢体定位（比如让脚精确踩在斜坡上），可以使用 `SkeletonModification2DTwoBoneIK`。将 IK 目标点交给 `Marker2D` 或射线检测结果，骨骼链会自动解算姿态。

**3D 动画**的导入非常直接：从 Blender 等 DCC 工具导出 FBX/glTF 时勾选动画，导入 Godot 后，动画会自动收纳在生成场景的 `AnimationPlayer` 节点中。之后 `AnimationTree` 指向这个 `AnimationPlayer` 即可照常使用。

---

### 💎 总结：你的学习节奏

1. **先只用 `Tween`**：所有“代码里顺手做一下”的动效，都用它完成，熟悉插值、缓动和链式调用。
2. **再掌握 `AnimationPlayer`**：学会在动画面板里创建片段、添加属性轨道，并尝试**方法轨道**来把游戏逻辑事件绑定到时间线上。
3. **最后引入 `AnimationTree`**：从一个简单的 `idle/walk` 状态机开始，逐步加入 `BlendSpace2D` 处理方向移动。记住那条法则——**能用 `AnimationPlayer` 直接 `play()` 解决的，就不要用状态机**。

Godot 动画系统的设计哲学是**用可视化编辑器承载“编排”，用代码控制“决策”**。当你把 `AnimationPlayer` 当作素材库、`AnimationTree` 当作决策层时，两者的分工就会变得非常清晰。