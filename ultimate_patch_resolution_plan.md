# 终极执行计划：猴子补丁 (Monkey Patch) 全景治理与架构升维 (v3.0 Ultimate)

本文档综合了多个维度的深度源码分析与架构洞察，提取了“AST 驱动的管线展平”、“AOP 中间件标准化”、“Worker 序列化根治”与“分级治理”等最佳实践，为您提供一份**最安全、最全面、可落地的单文件 HTML 重构计划**。

> **⚠️ 执行约束**：本文档仅为执行蓝图与验证口径。**只有在您明确回复“确认/点头”后**，我才会生成具体的修改清单并开始改动 HTML 代码。整个过程**保证不破坏单文件 HTML 的分发特性**。

---

## 1. 核心架构痛点与盲点诊断 (Diagnosis)

经过综合分析，当前基于 `TU.PatchRegistry` 和散落 `<script>` 块的补丁系统，不仅仅是“代码丑陋”，更在 V8 引擎底层和游戏生命周期埋下了致命隐患：

1. **原型链污染与 V8 性能雪崩 (Deoptimization)**
   - 痛点：`Renderer.applyPostFX` 被天气、水下雾气等包了 4 层洋葱模型；`Canvas getContext` 被直接全局劫持。
   - 盲点：过度包装破坏了 JIT 内联缓存（IC），导致高频渲染方法性能抖动。全局拦截 Canvas 更是会污染第三方库和后续 UI。
2. **多线程注入的工程死局 (Worker Stringification)**
   - 痛点：`tu_world_worker_patch_v1` 使用 `fn.toString()` 提取主线程类方法拼接 Worker 源码。
   - 盲点：闭包上下文丢失，且极度排斥未来的代码混淆（Minification）；并发 `generate` 存在竞态，旧请求会被静默挂起。
3. **初始化时序的隐式竞态 (Lifecycle Race Conditions)**
   - 痛点：`game:init:post` 被原始代码和补丁 `tu_experience_optimizations_v3` **重复触发两次**，且跨越了 `RAF`（渲染帧）启动前后。
   - 盲点：导致订阅者（如机器索引、Worker 光照同步）被重复调用，造成首帧严重卡顿尖峰。
4. **防御性吞噬导致的“静默失败” (Aggressive Error Swallowing)**
   - 痛点：`TU.Safe.run` 吞噬了大量同步异常，且对 `async/Promise` 拒绝无效；`PatchManager.once` 在 `async` 场景下会“假成功”。
   - 盲点：导致“存档写入失败却提示成功”、“幽灵方块”等致命 Bug 难以排查。

---

## 2. 核心重构战略：分级治理与 AOP 升维 (Tiered Strategy)

我们将摒弃“简单粗暴合并”，采用**分级治理（Tiered Governance）**，逐步将“猴子补丁”转化为正规的插件与事件驱动模型。

### 级别 1：完全原生化 (Native Integration) - 针对简单逻辑
- **对象**：`InputManager`, `AudioManager`, `TouchController` 的防卡死、音频交互补丁。
- **策略**：直接将补丁逻辑吸收（Merge）进原始类的 ES6 `class` 定义中，彻底删除 `__tuAudioVisPatch` 等防重入标志位和 `_oldXxx.call(this)` 的闭包开销。

### 级别 2：洋葱模型解构与管线化 (Pipeline Deconstruction) - 针对高频渲染
- **对象**：`Renderer.applyPostFX`, `Canvas getContext` 拦截, 天气系统的 `WeatherCanvasFX`。
- **策略**：
  - **渲染管线展平**：废除 4 层原型包装，引入标准的 `Renderer.postFxPipeline = []`。按物理逻辑严格排序：`SpeedBlur -> Bloom -> BaseAtmosphere -> Weather(Tint+Lightning) -> Underwater`。
  - **Canvas 局部化**：撤销对原生 `HTMLCanvasElement.prototype` 的全局劫持，改为仅在游戏主 Canvas 创建时进行局部的 `CanvasOptimizer` 代理。

### 级别 3：事件驱动与策略插件化 (Event-Driven & Plugins) - 针对巨型系统
- **对象**：`SaveSystem` 的 IDB 降级（600行）、`WorldWorkerClient`、酸雨危害系统。
- **策略**：
  - **存储适配器**：将 `SaveSystem` 的补丁抽离为标准的 `StorageAdapter`，统一处理 `localStorage -> IndexedDB` 的配额溢出与降级，补齐 `async` 错误边界。
  - **Worker 策略化**：废弃 `toString()` 拼接。采用子类化 (`PatchedWorldGenerator`) 或在 HTML 顶部统一定义 Worker 模块字符串，建立带版本号的 `WW_PROTOCOL` 通信机制，处理并发 `reject/queue` 竞态。

---

## 3. 分阶段执行计划 (Execution Phases)

为了控制风险，我们分为 5 个阶段执行，每阶段都具备**可独立回滚**和**可观测**的特性。

### 阶段零：基线与遥测 (Baseline & Telemetry)
*在改动任何核心逻辑前，先铺设安全网。*
- 修复 `PatchManager.once`：增加三态机，支持同步失败重试，修复 `async` Promise reject 时的“假成功”漏洞。
- 修复 `TU.Safe.run`：新增 `runAsync`，并将捕获的异常（含 swallowed 标记）接入现有的 `TU_Defensive.ErrorReporter`，不再静默吞噬。

### 阶段一：清理级别 1 补丁与消除时序毒瘤
- 将 `experience_optimized_v2`、`batching_idb_pickup_safe_v2`（部分）、`tu_experience_optimizations_v3`（输入复位部分）等直接合并到原始类中。
- **重构 `game:init:post`**：消除双重触发。拆分为精确的 `game:init:preRaf` 和 `game:init:postRaf`，修复机器索引和光照同步的性能尖峰。

### 阶段二：重塑渲染管线与解除 Canvas 劫持
- 撤销 `HTMLCanvasElement.prototype.getContext` 的全局劫持，收敛至安全的 `CanvasOptimizer`。
- 整合 5 个零散的天气补丁（基线、音频同步、特效优化、雨天修复、酸雨）：
  - 建立单一事实源 `game.weather` (包含 `acid` 状态)。
  - 将 `applyPostFX` 的多层劫持展平为顺序执行的 `switch/case` 或 Pipeline 数组，消除 `postA/lightning` 临时置零的 Hack 逻辑。

### 阶段三：存储降级适配与 Worker 通信重构
- **SaveSystem 降级**：将 IndexedDB 备份逻辑封装为标准的适配器，解决读写过程中的异常吞噬与“僵尸存档”问题。
- **WorldWorker 重构**：消除并发 `generate` 导致的 Promise 悬挂；建立包含 `protocolVersion` 的握手协议；补齐 `beforeunload` 时对 `pending Promise` 的 reject 清理。

### 阶段四：移除补丁引擎与终极清理
- 当所有补丁都被吸收进主源码或转为标准插件后，彻底删除 `TU.PatchRegistry`、`TU.Utils.safeApplyPatch` 及相关底层支持代码。
- 删除遗留的冗余 CSS 和无用的 `#weatherfx` 幽灵 DOM 挂载逻辑。

---

## 4. 严谨的验收清单 (Verification Checklist)

> **注**：以下验收将在执行阶段逐步进行。

- [ ] **性能指标 (Metrics)**：
  - `Renderer.applyPostFX` 调用栈层数 ≤ 1 层（无多层包装）。
  - 在固定场景（如下雨+酸雨切换）下，`requestAnimationFrame` 记录 60s 窗口的 p95/p99 帧耗时，不出现退化。
  - Chrome DevTools `--trace-deopt` 下核心热点函数无新增 Deopt 事件。
- [ ] **安全性与隔离 (Safety)**：
  - 控制台创建新的非游戏 `<canvas>`，其 `getContext('2d')` 返回原生对象，未被篡改。
  - 触发 IndexedDB 存储配额满溢或断电中断，存档不损坏，降级逻辑正常抛出 UI 错误而非静默吞噬。
- [ ] **一致性与竞态 (Consistency)**：
  - `game:init:post` 生命周期回调计数严格等于 1。
  - 短时间内连续触发 3 次 `Worker.generate`，前两次请求被正确 `reject` 或 `cancel`，第三次正常渲染，无 Promise 内存泄漏。

---

**请审阅上述终极版计划。**
如果您认可此方案的深度与严谨性，**请回复“确认”或“点头”**。
我将为您输出第一阶段的拟修改点清单，并在您二次确认后正式动刀修改 HTML。