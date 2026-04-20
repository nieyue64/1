# 终极重构执行计划：多维融合的猴子补丁治理方案 (The Ultimate Governance Plan)

> **版本：** v3.0 (融合了架构设计、风险控制、AST静态重写、性能诊断与容灾降级五大维度)
> **目标：** 彻底解决 `part3_game_single` 的“乱七八糟”猴补丁（Monkey Patch）乱象。在**绝对保证最终产物仍为单文件 HTML**的前提下，将脆弱的“运行时覆盖”升级为“可观测、可回滚、强解耦”的现代化事件与管线架构。

---

## 1. 五维剖析：为什么当前代码“乱七八糟”？
基于对之前各版本方案的深入提取，我们总结出现有代码面临的五大结构性危机（这也是重构的核心靶点）：

1. **不可控的洋葱劫持 (The Onion Model)** (`plan 1`, `plan 2` 贡献)
   - `Renderer.applyPostFX` 等热点方法被多个补丁层层包装（Wrapper），不仅性能严重劣化（去优化），且补丁间互相通过临时变量“压制”以保证视觉效果，逻辑极度脆弱。
2. **高危平台级与多线程污染** (`resolve_plan`, `plan 3` 贡献)
   - 补丁甚至覆写了 `HTMLCanvasElement.prototype.getContext`，导致全局 API 异变。
   - Worker 线程代码通过 `fn.toString()` 拼接，丢失了主线程闭包上下文，且极度不兼容未来的代码压缩混淆。
3. **静默错误与存储灾难** (`plan 3`, `resolve_plan` 贡献)
   - 随处可见的全局 `try-catch` 疯狂吞噬致命异常。当底层 ID 越界或 IndexedDB 删除报错时，程序不崩溃但导致“幽灵穿模”和“僵尸坏档”。
4. **硬编码与 ID 漂移** (`plan 3` 贡献)
   - 方块 ID 盲猜分配、生成群系硬编码。只要调整了补丁加载顺序，旧存档的数据就会因为“ID 漂移”而大面积损坏。
5. **受控与非受控补丁混用** (`resolve_plan` 贡献)
   - 一半的补丁走 `PatchRegistry.installAll()`（有排序），另一半在加载时立即覆盖原型（无序、不可逆）。两者混用导致真实执行顺序如同黑盒。

---

## 2. 核心治理原则 (The Refactoring Constraints)

为了保证游戏核心玩法与性能不丢失，执行时必须恪守以下底线：
- **交付底线**：**必须保持单文件 HTML**。所有重构与重组操作在中间态可拆分，但最终产物必须合并为一个 `index.html`，无需打包工具即可双击运行。
- **可逆性与观测性**：对每一处 API 的修改，必须具备 `dispose/unpatch` 的能力，并能在控制台追踪安装顺序。
- **单点击破**：每次仅重构一条链路（如：只做后处理管线、只做存储系统），必须经过“运行 -> 确认”的基线测试门禁才能进行下一项。

---

## 3. 分级执行路径 (Phased Execution Action Plan)

我们将采用“渐进式融合”策略，从风险最低的“基建与错误兜底”入手，最终实现“彻底剥离”。

### 阶段零：基线建立与安全网部署 (Safety Net & Baselines)
*目标：在不改变任何现有逻辑的前提下，建立异常监测与性能基准。*
1. **建立补丁台账**：扫描所有补丁，统一使用唯一的幂等 key（如 `patch:${id}`），区分出“受控”与“非受控”补丁。
2. **拆除“无声炸弹”**：移除全局吞噬异常的 `try-catch`。引入 `GlobalErrorBoundary` 和 `runAsync`，确保 Worker 挂起、音频节点抢占、存档失败等异常能显式抛出到控制台，停止带病运行。
3. **采集性能基线**：利用现有 logger 记录 `applyPostFX` 和 `WorldGenerator.generate` 的 p95/p99 帧耗时，作为重构后的验收门槛。

### 阶段一：核心管线扁平化与高危 API 隔离 (Pipeline Flattening & API Isolation)
*目标：消灭层层包裹的洋葱模型，规范化系统级劫持。*
1. **渲染后处理管线化**：
   - 彻底废除对 `Renderer.applyPostFX` 的多重 `_old.call(this)` 覆写。
   - 引入 `Renderer.postFxPipeline = []`。将基础特效、天气（Tint/Lightning）、水下迷雾（Underwater Fog）按照最终视觉顺序，展平为单层、线性的管线节点。
2. **平台级 Canvas API 隔离**：
   - 针对 `HTMLCanvasElement.getContext` 的强行覆盖，提取为独立的 `CanvasOptimizer` 模块。限制其只在游戏画布作用域内生效，严格保留现有“属性去重与缓存同步”语义，并提供可回滚开关。
3. **多线程注入加固**：
   - 梳理 `WorldWorker` 中的 `fn.toString()` 字符串拼接与正则替换。提取常量模板，并为 `__TU_OUT_POOL__` 内存复用机制增加正则匹配失败的回退校验，防止静默崩溃。

### 阶段二：生命周期重组与事件驱动 (Lifecycle & Event-Driven Migration)
*目标：将散落的“补丁覆盖”转化为“标准事件监听”。*
1. **收敛初始化事件**：修复 `Game.init` 被反复包装导致的 `game:init:post` 重复触发风险。统一生命周期钩子规范。
2. **状态驱动的解耦**：将天气逻辑（音效同步、视觉层覆盖）从各个原型方法剥离，统一监听 `window.GameEvents`（如 `on('weather:changed')`）。
3. **UI 交互与输入正规化**：将 `TouchController` 的防冲突补丁和 `InventoryUI` 的拖拽强化，吸收入原始类的原生方法中（**完全原生化**），不再作为外部补丁存在。

### 阶段三：持久化容灾与数据烘焙 (Storage Resilience & Data Baking)
*目标：彻底解决僵尸坏档与 ID 漂移问题。*
1. **存储适配器 (Storage Adapter)**：
   - 将 `SaveSystem` 高达 600 行的补丁封装为标准的 `IDBSavePlugin`。
   - 规范 `LocalStorage -> IndexedDB -> Lite` 的降级策略流，由适配器在异步边界内统一处理 Quota 异常，拒绝通过原型劫持层层覆盖。
2. **注册表与 ID 映射 (Registry & Palette Mapping)**：
   - 为世界生成的 DLC（新群系、新方块）引入 `BlockRegistry`。
   - 运行时进行“数据烘焙（Data Baking）”写入底层高性能 `TypedArray`，并在存档头部加入 ID 映射表，确保增删补丁永远不会导致旧存档方块错乱。

---

## 4. 自动化与人工验收标准 (Validation & Acceptance Criteria)

每完成一个阶段的改写（例如阶段一的管线化），必须通过以下严格验收，才能继续：

1. **架构验收**：`Renderer.applyPostFX` 和 `WorldGenerator` 不再存在多层嵌套的闭包包装（可通过输出 wrapper 层数检验）。所有受控补丁可通过 `dispose()` 卸载。
2. **功能验收（无头验证）**：在本地 HTTP 服务启动下，能够正常生成世界、渲染一帧（包含水雾与天气）、触发一次 IndexedDB 存档读写。
3. **性能不回退验收**：核心 Tick Time 和 Render Time 的长帧（>50ms）发生率，不得高于阶段零测得的基线数据。
4. **错误清零**：控制台无任何未捕获的致命错误（Fatal）与 Promise Rejection。

---

> **执行声明：** 本计划在保证极致性能与功能完整的前提下，吸收了 AST 静态分析的确定性、AOP 的解耦性、事件驱动的灵活性以及系统容灾的鲁棒性。**一切修改最终均在当前 HTML 内完成闭环。**
