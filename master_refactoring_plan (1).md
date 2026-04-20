# 终极重构计划：基于分级治理与管线化的猴子补丁消除方案 (Master Plan)

综合了 5 份独立 AI 深度分析报告的优势，剥离了其中不切实际的方案（如引入 Webpack 破坏单文件约束、盲目依赖 AST 带来的执行风险），我们提炼出了一套**最务实、最安全、最彻底**的重构计划。

本计划的核心思想是：**停止“用魔法打败魔法”（套娃包装），转向“分级治理”与“显式架构”。**

---

## 1. 核心约束与设计基准 (Constraints & Baselines)
1. **单文件交付约束**：重构后的产物必须依然是独立的 HTML 文件，不引入外部构建工具（如 Vite/Webpack）。
2. **消除 JIT 性能劣化**：彻底终结多层 `_old.apply(this)` 造成的调用栈过深和 V8 引擎去优化（Deoptimization）。
3. **拒绝静默失败**：修复现存补丁中 `try-catch` 强行吞噬致命异常（如方块越界、IDB 写入失败）的“掩耳盗铃”行为。

---

## 2. 核心重构策略：分级治理 (Tiered Governance)

我们将现有 40+ 个“乱七八糟”的补丁，按照其复杂度分为三个级别，采取完全不同的治理手段：

### 级别 1：扁平化原生合并 (Native Integration)
**针对**：仅为了挂载事件或修改单一逻辑的简单补丁（如 `InputManager`, `AudioManager`, `TouchController`）。
- **行动**：直接将补丁逻辑吸收入原始 ES6 `class` 定义中。
- **优化**：彻底删除 `__xxxInstalled` 等防御性全局变量；将依赖原始执行流程的钩子，改为监听 `window.GameEvents`（例如直接在类内监听 `game:init:post`，而不是去重写 `Game.init`）。

### 级别 2：洋葱模型解构与显式管线化 (Onion Deconstruction & Pipelining)
**针对**：被多个补丁反复嵌套覆盖的高频热点方法（如 `Renderer.prototype.applyPostFX`、`Game.prototype._handleInteraction`）。
- **行动**：**绝对禁止再次包裹（Wrap）**。我们将手动展平（Unroll）这 3~4 层嵌套。
- **优化**：引入**显式渲染管线（PostFxPipeline）**。将散落的酸雨、雷暴、水下迷雾特效，改造为按顺序注册的 `pass` 节点（`speedBlur -> bloom -> baseAtmosphere -> weather -> underwater`），由渲染器一次性线性执行，消除互相覆盖的风险。

### 级别 3：策略抽象与适配器提取 (Strategy & Adapter Extraction)
**针对**：长达数百行、逻辑极其复杂的系统级巨型补丁（如存档降级、Web Worker 源码注入）。
- **行动 1（存储层）**：将 `SaveSystem` 中复杂的 LocalStorage/IndexedDB 回退和 50k 差异降级逻辑，抽离为一个独立的 `StorageAdapter` 类，规范化异步边界（Async Boundaries），不再通过覆写 `save()` 方法来实现。
- **行动 2（多线程层）**：废弃极其脆弱的 `fn.toString()` 字符串正则替换拼接。引入 `PatchedWorldGenerator extends WorldGenerator`（子类化策略），在 Worker 中直接实例化子类，保证代码压缩兼容性。
- **行动 3（平台层）**：将高危的 `HTMLCanvasElement.prototype.getContext` 平台级拦截，封装为独立的 `CanvasOptimizer.install()` 模块，限定作用域仅针对游戏画布，并保留原有的属性去重和状态同步语义。

---

## 3. 具体执行步骤 (Execution Roadmap)

### 阶段零：建立补丁台账与安全网 (Ledger & Safety Net)
1. 区分“受控补丁（走 `PatchRegistry`）”与“非受控补丁（IIFE 直接执行）”。
2. 修复 `Safe.run` 吞噬异步 Promise 异常的问题，确保重构期间任何逻辑错误都会在控制台显式报错。
3. 剔除 `Game.init` 中导致 `game:init:post` 重复发射（Double-emit）的冗余补丁。

### 阶段一：收复原生类与事件驱动 (Tier 1 Execution)
1. 将 UI（拖拽优化）、音频（雨声合成器）和输入（防误触安全区）相关的补丁，直接合并到原始类定义。
2. 清理超过 15 个无用的 `__xxxPatched` 标志位。

### 阶段二：重塑渲染管线与核心循环 (Tier 2 Execution)
1. 重写 `Renderer.prototype.applyPostFX`，剥离所有 `_old` 引用，植入按顺序执行的后处理队列。
2. 收敛天气系统（WeatherSystem），将其作为唯一的事实数据源，取代各处零散的天气覆盖逻辑。
3. 整合 `_writeTileFast` 和 `markTile` 逻辑，改用 `block:written` 事件触发区块重绘与 Worker 同步。

### 阶段三：治理底层炸弹 (Tier 3 Execution)
1. **重构 SaveSystem**：接入 `StorageAdapter`，理顺自动保存与降级模式的状态机。
2. **重构 TileLogicEngine / Worker**：将流体模拟、`__TU_OUT_POOL__` 内存复用池和 `Uint32Array` 数据压缩打包协议，转化为硬编码的常量模板或标准子类注入，彻底消灭 `replace()` 带来的正则脆弱性。

### 阶段四：验证与对齐 (Validation)
- **帧率监控**：确保 JIT 优化恢复后，渲染帧耗时（Frame Time）p99 指标不回退。
- **链路测试**：测试世界生成、存档写入（强制触发配额溢出测试 IDB 降级）、触发一次酸雨天气，验证新架构下的逻辑连贯性。

---

> **协同优势说明**：此方案吸收了计划 2 的“异常治理”，计划 3 的“台账与安全管线”，计划 1 的“子类/适配器抽象”，以及原始计划的“事件驱动替代”，构成了当前环境下风险最低、收益最大的“治本”蓝图。