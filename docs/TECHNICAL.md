# DirectionalGravity 技术文档 
> 引擎：**Unreal Engine 4.26** | 语言：**Blueprint（100%）** | 物理：**PhysX**

---

## 1. 项目概览

| 项 | 值 |
|----|-----|
| 项目名 | DirectionalGravity（DirectionalGravityforPhys） |
| 类型 | 3D 物理平台跳跃 Demo |
| 默认地图 | `/Game/DirectionalGravity/Maps/GravityMap` |
| GameMode | `DirectionalGravityGameMode` |
| 核心机制 | 区域化方向重力：玩家在不同重力场中体验定向/径向引力，角色与相机随重力对齐 |
| 发布 | Windows 64-bit Development 包（2025.05.13） |

---

## 2. 目录与资源结构

```
DirectionalGravityforPhys/
├── DirectionalGravityforPhys.uproject
├── Config/                          # 引擎/输入/编辑器配置
├── Content/                         # 当前副本缺失，完整工程联系作者...
│   ├── DirectionalGravity/          # ★ 核心自研内容
│   │   ├── Blueprints/
│   │   │   ├── DirectionalGravityGameMode
│   │   │   ├── GravityLogic/        # 重力系统 + 角色
│   │   │   ├── SceneObjects/        # 重力场 + 交互物
│   │   │   └── Helpers/             # 数学库 + 物理材质
│   │   ├── Maps/                    # GravityMap, DemoMap
│   │   ├── Meshes/                  # SM_Cube, SM_Planet, SM_Hemisphere, map-part
│   │   ├── Materials/
│   │   └── Mannequin/               # 第三人称动画资源
│   ├── Blueprints/                  # Tube 关卡系统（早期/并行内容）
│   ├── Sch/                         # UI、相机、天空盒
│   └── StarterContent/              # UE 模板资源（非自研）
├── Saved/ / Intermediate/ / Build/  # 构建产物
└── WindowsNoEditor/                 # 打包输出
```

---

## 3. 核心架构

### 3.1 方向重力系统总览

```
┌─────────────────────────────────────────────────────────┐
│                    World (GravityZ = 0)                  │
│  ┌──────────────────┐    Overlap/Tick    ┌────────────┐ │
│  │ BP_GravityVolume │ ─────────────────► │ BP_Gravity │ │
│  │ Base             │   提供方向/强度     │ Receiver   │ │
│  │  ├─ Box          │                    │ (Component)│ │
│  │  └─ Spherical    │                    └─────┬──────┘ │
│  └──────────────────┘                          │       │
│                                                 ▼       │
│                              ┌─────────────────────────┐│
│                              │ BP_GravityCharacterBase ││
│                              │  ├─ FP (第一人称)        ││
│                              │  └─ TP (第三人称)        ││
│                              └─────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

**设计要点：**

1. **关闭世界重力**：`Config/DefaultEngine.ini` 中 `DefaultGravityZ=0`，避免与自定义重力冲突  
2. **Volume 负责「场」**：定义空间范围与重力计算规则  
3. **Receiver 负责「接收」**：挂在可受重力影响的 Actor 上，汇总当前生效的重力  
4. **Character 负责「响应」**：移动、跳跃、地面检测、Rotation 对齐均基于 Receiver 提供的 Up/Gravity 向量  

### 3.2 关键 Blueprint 清单

#### 重力系统（GravityLogic + SceneObjects）

| Blueprint | 类型 | 职责 |
|-----------|------|------|
| `BP_GravityVolumeBase` | Actor | 重力场基类；Tick/Overlap 逻辑；暴露 Gravity 属性组 |
| `BP_GravityVolumeBox` | Actor | 盒形区域，提供**固定方向**重力（如局部 -X、-Y） |
| `BP_GravityVolumeSpherical` | Actor | 球形区域，提供**径向**重力（指向/背离球心） |
| `BP_GravityReceiver` | **ActorComponent** | 检测所在重力场，输出当前 `GravityDirection`、`GravityStrength` |
| `BP_GravityCharacterBase` | Character/Pawn | 重力对齐移动、GroundDetection、Turning、Actions |
| `BP_GravityCharacterFP` | Character | 第一人称：Camera 挂载在 Capsule 上 |
| `BP_GravityCharacterTP` | Character | 第三人称：Mannequin Mesh + ThirdPerson_AnimBP |
| `DirectionalGravityGameMode` | GameMode | 默认 GameMode，设置默认 Pawn |

#### 工具与物理

| Blueprint | 类型 | 职责 |
|-----------|------|------|
| `BP_MathHelpers` | Blueprint Function Library | 向量投影、重力→Rotation、角度插值等 |
| `PM_CharacterWalk` | Physical Material | 正常行走摩擦/反弹 |
| `PM_CharacterSlide` | Physical Material | 低摩擦滑行表面 |
| `BP_InteractiveObject` | Actor | 可交互物理物体 |

#### 关卡系统

| Blueprint | 职责 |
|-----------|------|
| `BP_TubeBase` | 管道关卡段基类 |
| `BP_TubeA` ~ `BP_TubeF` | 6 种管道段变体 |
| `BP_TubeSpawner` | 按 `TubeClass` 动态生成管道段 |
| `BP_NextBox` | 关卡进度/切换触发 |

#### UI

| 资产 | 职责 |
|------|------|
| `UMG_MainUI` | 主界面/HUD |
| `UMG_GameOver` | 游戏结束界面 |

---

## 4. 核心机制详解

### 4.1 为什么关闭 DefaultGravityZ？

UE 默认只对 RigidBody 施加 `-Z` 方向重力。方向重力需要：

- 重力方向随区域变化（甚至每帧变化，如球形场）
- 角色 Controller 的「Up」不再等于 `FVector(0,0,1)`
- 移动输入需投影到**重力切平面**，而非 XY 平面

因此设置 `DefaultGravityZ=0`，改由 Blueprint 每帧对 Character 施加 `AddForce` / 自定义 `Velocity` 变更。

### 4.2 盒形重力场（Box）

**算法：**

```
OnOverlapBegin(Receiver):
    RegisterVolume(this)

Each Tick:
    if Receiver inside Box bounds:
        GravityDir = Box.GetGravityDirection()  // 固定单位向量，如 -Forward
        Strength = Box.GravityStrength
```

### 4.3 球形重力场（Spherical）

**算法：**

```
Each Tick for Receiver at Position P:
    Center = Sphere.GetActorLocation()
    GravityDir = (Center - P).Normalize()   // 吸向球心（行星重力）
    // 或 (P - Center).Normalize()          // 斥力/反重力
    Strength = f(distance)                    // 可选：距离衰减
```

**典型用途：** 小行星表面行走、环形关卡

### 4.4 Receiver 组件工作流程

```
1. BeginPlay: 初始化 DefaultGravity（如 -WorldZ 或零）
2. OnComponentBeginOverlap / Tick:
   - 收集所有重叠的 GravityVolume
   - 按优先级/距离/体积选择生效 Volume（具体策略需在完整工程中确认）
   - 更新 CurrentGravityDirection, CurrentGravityStrength
3. 对外提供接口:
   - GetGravityDirection() → FVector
   - GetUpVector() → -GetGravityDirection()
   - GetGravityStrength() → float
```
### 4.5 重力对齐角色控制器

`BP_GravityCharacterBase` 编辑器中可见的功能分区：

| 模块 | 功能 |
|------|------|
| **Moving** | 将 WASD 输入投影到垂直于重力的平面；沿 `WalkAcceleration` 加速 |
| **GroundDetection** | 沿 `-Up` 方向 SphereTrace/LineTrace 检测地面 |
| **Actions** | Jump：沿 `-GravityDirection` 施加冲量 |
| **Camera** | FP/TP 各自处理 CameraBoom 或 FP Camera 相对 Up 的旋转 |

**地面检测伪代码：**

```
Up = Receiver.GetUpVector()
Start = CharacterLocation
End = Start - Up * GroundCheckDistance
Hit = LineTrace(Start, End, GroundChannel)
bIsGrounded = Hit.bBlockingHit
GroundNormal = Hit.Normal
```

**Rotation 对齐：**

```
TargetRotation = MakeRotFromZX(Up, ForwardProjectedOnPlane)
Character.SetActorRotation(RInterpTo(Current, Target, DeltaTime, InterpSpeed))
```

### 4.6 移动输入投影

```
InputForward, InputRight = 来自 Enhanced/Legacy Input
GravityUp = Receiver.GetUpVector()
WorldForward = Character.GetActorForwardVector()
// 投影到重力切平面
TangentForward = (WorldForward - GravityUp * Dot(WorldForward, GravityUp)).Normalize()
TangentRight = Cross(GravityUp, TangentForward)
MoveDelta = TangentForward * InputForward + TangentRight * InputRight
```

### 4.7 物理材质（Walk vs Slide）

- `PM_CharacterWalk`：高摩擦，正常平台
- `PM_CharacterSlide`：低摩擦，滑行管道/斜面

通过 `PhysicalMaterial` 的 `Friction` 属性影响 `CharacterMovement` 或 `AddForce` 模式下的滑动行为。

### 4.8 Tube 模块化关卡

```
BP_TubeSpawner:
    TubeClass = BP_TubeA / B / C ...   // 可配置
    SpawnNextSegment():
        SpawnActor(TubeClass, SocketLocation)
        Attach or align to previous segment socket
```

**设计价值：** 关卡设计师只需拼 Spawner + 换 TubeClass，无需每次手动摆放。

---

## 5. 配置要点

### DefaultEngine.ini

```ini
[/Script/EngineSettings.GameMapsSettings]
EditorStartupMap=/Game/DirectionalGravity/Maps/GravityMap
GameDefaultMap=/Game/DirectionalGravity/Maps/GravityMap
GlobalDefaultGameMode=/Game/DirectionalGravity/Blueprints/DirectionalGravityGameMode

[/Script/Engine.PhysicsSettings]
DefaultGravityZ=0.000000   # ★ 关键：关闭默认重力
```

### DefaultInput.ini（主要绑定）

| 输入 | 键位 |
|------|------|
| MoveForward | W/S |
| MoveRight | A/D |
| Turn | I/K/J/L |
| Jump | Space |

---

## 6. 设计模式总结

| 模式 | 项目中的体现 |
|------|-------------|
| **组件模式** | `BP_GravityReceiver` 可挂到任意 Actor |
| **继承** | VolumeBase → Box/Spherical；CharacterBase → FP/TP |
| **策略模式** | 不同 Volume 类型提供不同重力计算，Character 无感知 |
| **函数库** | `BP_MathHelpers` 集中数学逻辑，避免 Graph 重复 |
| **数据驱动** | TubeSpawner 的 `TubeClass` 配置化关卡段 |
---

## 8. 与同类游戏/方案对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| **本项目 Volume+Receiver** | 模块化、关卡可摆放、易扩展 | 需自研 Character 逻辑 |
| 仅改 `AddForce(-Z)` | 简单 | 无法方向重力 |
| 旋转整个 World/Level | 概念简单 | 性能差、影响所有 Actor、相机难处理 |
| UE5 `GlobalGravity`（如有插件/自定义） | 引擎级 | UE4.26 无开箱方案 |

---
