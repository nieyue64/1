# 终极全知重构蓝图：猴子补丁 (Monkey Patch) 治理与架构飞升计划 (The Omni-Plan)

> **前言**：本计划萃取了 6 份独立 AI 深度架构报告的精华。我们解决了各方案间的冲突（如“全面 AST 重写的高风险”与“人工梳理的低效”），保留了“单文件绝对约束”，融合了“V8 性能诊断”、“多线程沙盒容灾”、“AOP 管线解构”与“防静默错误门禁”，为您呈现当前**最务实、最安全、最彻底**的终极执行方案。

---

## 1. 战略共识与核心约束 (Strategic Consensus & Constraints)

在正式执行前，我们确立三项不可动摇的底线，确保重构过程不会带来灾难：
1. **物理形态约束（来自方案 1、2、5）**：最终产出**必须是单一可运行的 HTML 文件**。不引入 Webpack/Vite 等重型构建工具破坏分发特性，所有改动在文件内部进行。
2. **重构手段平衡（化解 AST 冲突）**：**放弃盲目的全自动 AST 替换**，改用“AST 辅助生成补丁拓扑图与依赖链”，配合“定向人工/半自动安全重写”。既保证依赖顺序 100% 正确，又规避语法树回写破坏代码结构的风险。
3. **零静默退化约束（来自方案 3、4）**：必须先建立“错误暴露机制”与“帧耗时（p95/p99）基线”，重构后的版本在相同场景下的性能与稳定性只能持平或更好。

---

## 2. 深度诊断：五大核心危机 (The 5 Core Crises)

结合 6 份报告的扫描，代码当前的“乱”并非单纯的风格问题，而是系统级危机：
1. **V8 性能雪崩 (Deoptimization)**：`Renderer.applyPostFX` 等热点方法被“洋葱模型”嵌套劫持了 4 层，破坏了 JIT 内联缓存。
2. **多线程注入的工程死局**：`WorldGenerator` 使用 `fn.toString()` 提取主线程闭包拼接 Worker，极度排斥代码压缩，且并发 `generate` 存在竞态挂起风险。
3. **静默失败与状态损坏 (Silent Failures)**：大量 `try-catch` 与 `Safe.run` 吞噬异常。导致“方块 ID 越界（幽灵方块）”与“IndexedDB 写入失败（僵尸存档）”不抛出错误。
4. **生命周期竞态 (Lifecycle Race Conditions)**：原始代码与优化补丁重复触发 `game:init:post`，导致首帧卡顿；渲染管线中后装补丁通过临时置零等 Hack 压制前装逻辑。
5. **平台级 API 污染**：全局粗暴劫持了 `HTMLCanvasElement.prototype.getContext`，随时可能与后续引入的第三方 UI 库冲突。

---

## 3. 分级治理架构设计 (Tiered Governance Architecture)

我们摒弃“一刀切”的重写，采用“分级治理（Tiered Governance）”，将 40+ 个补丁根据耦合度实施不同的降维打击：

### 级别 1：完全原生化 (Native Integration)
*   **针对**：仅需附加在原逻辑上的孤立修补（如 `InputManager` 防卡死、`AudioManager` 音效、`TouchController` 摇杆）。
*   **动作**：直接将其逻辑**融合进**原始 ES6 `class` 定义中。彻底拔除 `__tuAudioVisPatch` 等数十个防重入标志位与闭包开销。

### 级别 2：事件驱动替代 (Event-Driven Migration)
*   **针对**：被用来挂载旁路逻辑的生命周期劫持（如在 `markTile` 后触发红石更新）。
*   **动作**：废除对核心方法的覆盖，改为监听底层的 `window.GameEvents`。彻底修复 `game:init:post` 的重复发射问题，将旁路逻辑（如机器索引、环境音响）解耦。

### 级别 3：管线化与平台隔离 (Pipelining & Isolation)
*   **针对**：`Renderer.applyPostFX`（渲染劫持）与 `Canvas.getContext`（API 劫持）。
*   **动作**：
    *   **渲染管线化**：禁止 `_old.apply(this)`。在 `Renderer` 中引入显式的 `PostFxPipeline`，将速度模糊、泛光、天气、水下迷雾按固定顺序展平为单层遍历。
    *   **Canvas 隔离**：将 `getContext` 劫持抽取为显式的 `CanvasOptimizer`，限定只代理游戏主画布，保留属性去重与状态同步语义，不污染全局环境。

### 级别 4：适配器抽象与数据烘焙 (Adapters & Data Baking)
*   **针对**：长达 600 行的 `SaveSystem` 降级逻辑与 Web Worker 源码注入、硬编码群系。
*   **动作**：
    *   **存储适配器**：将 IndexedDB 回退封装为 `StorageAdapter`，补齐异步边界（Async Boundaries），不再通过覆写 `save()` 实现，彻底解决配额溢出时的坏档问题。
    *   **Worker 策略化**：废弃 `toString()` 拼接。采用静态模块模板（Template String）与带版本号的 `WW_PROTOCOL` 标准通信，处理并发 `reject/queue`。
    *   **注册表**：建立 `BlockRegistry` 与存档 ID 映射表，解决补丁顺序变动导致“ID 漂移（坏档）”的顽疾。

---

## 4. 零风险演进路线图 (Zero-Risk Execution Roadmap)

整个重构分为 4 个严密的阶段，每阶段执行完毕均需通过“防退化门禁”验证。

### 🏁 阶段零：安全网铺设与拓扑分析 (Safety Net & Topology)
1. **AST 辅助拓扑**：分析所有 `AssignmentExpression`，理清现存补丁真实的覆盖顺序（如：水下滤镜必须在酸雨之后），建立**执行顺序链台账**。
2. **错误暴露**：重构 `TU.Safe.run`，引入 `runAsync` 和 `GlobalErrorBoundary`。停止吞噬错误，让隐藏的 Promise Rejection 和渲染异常显式抛出至控制台。
3. **基线采样**：记录重构前 `applyPostFX` 和 `generate` 的耗时（p50/p95/p99）。

### 🛠️ 阶段一：外围收复与事件解耦 (Tier 1 & 2 Execution)
1. 剥离并原生化所有外围 UI、音频、输入防冲突补丁（级别 1）。
2. 将 `_writeTileFast` 劫持和初始化的 Hack 代码改为事件监听（级别 2），消除生命周期竞态。
*   *验收：按键/触摸不失效，音效正常，控制台无事件循环死锁。*

### 🎨 阶段二：重塑管线与解除隔离 (Tier 3 Execution)
1. 展平 `applyPostFX` 4 层包裹，植入 5 阶段显式后处理队列。
2. 剥离 `Canvas.getContext` 污染，接入 `CanvasOptimizer`。
3. 建立统一的 `WeatherSystem`，接管散落的 5 个天气控制块。
*   *验收：帧耗时（p99）不可回退，Chrome DevTools 确认热点函数解除 Deopt。切换天气/入水视觉效果正确。*

### 💾 阶段三：根治底层炸弹 (Tier 4 Execution)
1. 将 IndexedDB 回退逻辑迁移至 `StorageAdapter`，明确异步异常边界。
2. 重构 Worker 通信协议与生成器子类化，消灭正则替换脆弱性。
3. 引入区块注册表与 ID 烘焙机制。
*   *验收：断开本地存储配额测试 IDB 降级，并发请求生成区块不挂起，存档读写 100% 成功。*

---

> **协同优势总结**：
> 本计划吸收了 AST 的**严谨拓扑**（方案 1）、AOP 的**事件解耦**（方案 2）、Worker 与存储的**容灾设计**（方案 3）、分级治理的**风险控制**（方案 4）、以及**性能去优化（Deopt）的底层诊断**（方案 5）。
> 这是目前针对当前代码库能够给出的最全面、最安全的终极重构方案。
