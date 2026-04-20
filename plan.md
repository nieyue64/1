# 重构执行计划：基于分级治理架构的猴子补丁优化 (v2.0 终极版)

经过对 `/workspace/part3_game_single (6) (9)(9)(2)(6).html` 源码长达数十次的并行深度分析与引擎时序验证，我们摒弃了“简单粗暴合并”的危险做法，制定了以下科学、安全的**分级治理规范（Tiered Governance Spec）**。

---

## 核心重构策略：分级治理 (Tiered Governance)

### 级别 1：完全原生化 (Native Integration)
**适用对象**：逻辑扁平、无嵌套、直接依赖闭包常量的补丁。
- **涉及类**：`InputManager`, `AudioManager`, `TextureGenerator`, `DroppedItemManager`, `TouchController`
- **操作规范**：
  - 将补丁中的新增方法（如 `_startRainSynth`, `beep`, `noise`）直接写入 ES6 `class` 内部。
  - 将针对原有方法的劫持（如 `_drawPixelArt`, `spawn`），直接合并到类内部的方法中。
  - **彻底移除**防重复执行标志（如 `__logicPatched = !0`、`__tuAudioVisPatch`）。
  - **彻底移除** `_oldXxx.call(this)` 的闭包开销，将代码扁平化为纯粹的面向对象调用。

### 级别 2：洋葱模型解构 (Onion Model Deconstruction)
**适用对象**：被多个独立补丁层层嵌套修改的核心高频方法（洋葱模式）。
- **涉及类与方法**：
  - `Renderer.prototype.applyPostFX` (被天气、水下雾气等修改了 4 次)
  - `Game.prototype.init` / `Game.prototype._startRaf`
  - `Game.prototype._writeTileFast` (被双重包装)
- **操作规范**：
  - **拒绝简单合并**。人工梳理这 3~4 层拦截逻辑，按其原本的执行顺序（Order），将它们**展平（Unroll）并融合为一个单一的、线性的完整方法**，然后替换原有的类方法。
  - 消除多层函数调用的性能损耗（Call Stack Overhead）。

### 级别 3：规范化插件架构 (Standardized Plugin Architecture)
**适用对象**：长达数百行、引入了全新子系统（如降级存储、多线程）的巨型补丁。
- **涉及系统**：
  - `SaveSystem` 的 IndexedDB 降级存储系统 (长达 600 行)
  - `WorldWorkerClient` 的离屏逻辑同步系统
  - 自动化机器（Machines）的全局索引系统
- **操作规范**：
  - **禁止合并入核心类**（防止类体积失控）。
  - 将这些独立逻辑提取为真正的**独立模块/类**（例如命名为 `IDBSavePlugin`, `WorkerSyncPlugin`）。
  - 依然利用现有的 `TU.PatchRegistry` 或 `GameEvents` 系统，在 `game:init:post` 或 `game:ready` 生命周期钩子中，以标准的“插件注入”方式挂载，保留代码的极度解耦优势。

---

## 执行步骤 (Action Plan)

1. **阶段一：安全备份与级别 1 重构**
   - 提取 `InputManager`, `AudioManager` 等类，进行级别 1 原生化合并，清除防御性变量，验证闭包作用域（`IDS.WIRE_ON` 等）。
2. **阶段二：级别 2 核心渲染管线展平**
   - 重点重写 `applyPostFX` 和 `_writeTileFast`，按物理逻辑顺序融合多重环境滤镜和状态分发器。
3. **阶段三：级别 3 巨型补丁插件化**
   - 隔离 `SaveSystem` 末尾的 600 行代码，封装为标准 Plugin 类，确保其注册流程清晰可见，不再是一坨无名的 IIFE 匿名函数。
4. **阶段四：测试与验证**
   - 开启本地 HTTP 服务 (`python3 -m http.server`)，利用 OpenPreview 验证游戏各项系统（渲染、声音、保存）是否稳定运行。

---
**此方案已通过严谨的变量作用域与时序验证，确保不会破坏游戏的任何现有功能。**