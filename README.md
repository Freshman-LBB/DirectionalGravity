# DirectionalGravity

**UE4 方向重力物理平台 Demo** — 在自定义重力场中行走、跳跃，支持墙面行走与球形径向引力。

| | |
|---|---|
| 引擎 | Unreal Engine **4.26** |
| 实现 | Blueprint（100%） |
| 物理 | PhysX |
| 默认地图 | `GravityMap` |
| Windows 打包 | 2025.05.13 |
---

## 核心玩法

- **方向重力**：关闭世界默认重力（`DefaultGravityZ = 0`），由重力场 Actor 向角色下发局部重力方向与强度。
- **两种重力场**：盒形（固定方向）、球形（径向指向/背离球心）。
- **重力对齐角色**：移动、跳跃、地面检测、相机旋转均跟随当前「Up」向量。
- **双视角**：第一人称 / 第三人称共用 BP_GravityCharacterBase 逻辑。
- **模块化关卡**：Tube 分段 + Spawner 动态生成管道式关卡。

### 架构示意

```
GravityVolume (Box / Spherical)重力体积（盒子/球形）
        │ Overlap / Tick   │重叠/重叠
        ▼
GravityReceiver (Component)GravityReceiver(组件)
        │
        ▼
GravityCharacterBase → FP / TP重力特征库&rarr； FP / TP
```

更完整的设计说明、答见 [docs/TECHNICAL.md](docs/TECHNICAL.md)。

---

## 环境要求

- **Unreal Engine 4.26**（需与 .uproject 中 EngineAssociation 一致）
- Windows 10/11（开发与打包目标平台）
- 建议显存 ≥ 4 GB

---

## 快速开始

1. 克隆仓库：
   ```bash   ”“bash
   git clone https://github.com/Freshman-LBB/DirectionalGravity.gitgit clone https://github.com/Freshman-LBB/DirectionalGravity.git
   cd DirectionalGravity
   ```
2. 双击 DirectionalGravityforPhys.uproject，在启动器中选择 **UE 4.26** 打开。
3. 首次打开会编译 Shader，等待完成后在编辑器中 **Play**。
4. 默认地图：/Game/DirectionalGravity/Maps/GravityMap。

### 操作说明

| 操作 | 按键 |
|------|------|
| 移动 | W / A / S / D || w / a / s / d |
| 跳跃 | Space |
| 交互 | I / K / J / L || I / k / j / l |

---

## 仓库结构

```
DirectionalGravityforPhys.uproject   # 工程入口
Config/                              # 引擎、输入、地图与 GameMode 配置
Content/   内容/
  DirectionalGravity/                # ★ 自研：重力系统、角色、关卡
  Blueprints/                        # Tube 关卡系统
  Sch/                               # UI、相机等
  StarterContent/                    # UE 模板资源（非自研）
docs/   文档/
  TECHNICAL.md                       # 技术文档（架构、面试要点）
```

> 本仓库**不包含** `WindowsNoEditor/` 打包目录（体积大、不适合 Git）。需要可执行文件时，请联系作者。
---

## 技术亮点

- 设计 **Volume + Receiver** 组件化方向重力框架，支持盒形 / 球形两种策略，角色逻辑与场类型解耦。
- 实现重力对齐移动、地面检测、跳跃与相机插值对齐，抽象 Base 并扩展 FP/TP。
- 封装 BP_MathHelpers 与 Walk/Slide 物理材质，完成 UMG 与 Windows 打包流程。

---

## 许可证

本项目为个人学习 / 作品展示用途。如需二次使用或商用，请联系作者。

---

## 作者
GitHub: [@Freshman-LBB] (https://github.com/Freshman-LBB)
