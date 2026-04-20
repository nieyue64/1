# 深度重构与架构优化计划 (Deep Refactoring & Architecture Plan)

**目标:** 彻底解决 `part3_game_single (6) (9)(9)(2)(6).html` 中猴子补丁（Monkey Patches）带来的架构混乱、性能隐患、竞态条件以及维护灾难。将侵入式的原型链劫持，平滑迁移至标准化的 AOP（面向切面编程）包装器和原生的事件驱动（Event-Driven）模型。

**架构评估与洞察 (Architectural Insights):**
经过 10 个独立 Search Agent 的全面源码深度扫描，我们发现了当前代码库的几个核心特质与痛点：
1. **庞大的寄生型框架**：代码中存在多达 23 处以 `__xxxInstalled` 命名的防重入标志，干涉了从 `Game`、`Renderer`、`SaveSystem` 到 `TileLogicEngine` (Worker) 的所有核心子系统。
2. **底层基础设施完备**：虽然补丁写得乱，但底层其实已经具备了极其优秀的隔离与调度机制——包括 `window.TU.PatchManager` (AOP 包装与遥测)、`TU.PatchRegistry` (依赖排序与延迟加载)、`window.AppServices` (服务定位器) 以及 `window.GameEvents` (全局事件总线)。
3. **滥用 Monkey Patch**：许多仅仅是为了在生命周期特定阶段（如渲染后、初始化后、方块放置后）追加一段独立逻辑的补丁，完全可以（也应该）被 `window.GameEvents` 事件监听器替代。
4. **硬核的 Worker 注入**：通过 `Function.prototype.toString()` 和正则替换，动态向 Web Worker 源码注入流体/电路模拟器和内存池（Zero-allocation Buffer Recycling），这部分必须极其谨慎地保留或标准化。

---

## 阶段一：补丁标准化与 AOP 迁移 (Patch Standardization)

**策略:** 废弃所有手动编写的 `if (!Class.prototype.__xxxInstalled)` 闭包，统一接入底层的 `TU.PatchManager.wrapProto`。

- [ ] **Task 1.1: 渲染器与核心循环的 AOP 改造**
  - **目标:** 重构 `Renderer.prototype.renderWorld`、`Renderer.prototype.invalidateTile` 以及 `GameClass.prototype.tick`。
  - **行动:** 移除手动缓存的 `_old` 函数引用。使用 `PatchManager.wrapProto` 重新包裹区块批处理 (`__chunkBatchSafeInstalled`) 相关的逻辑。

- [ ] **Task 1.2: 存档与 IndexedDB 降级逻辑的 AOP 改造**
  - **目标:** 重构 `SaveSystem.prototype.save` 和 `tickAutosave`。
  - **行动:** 使用 `wrapProto` 将 `localStorage` 配额溢出回退至 `IndexedDB`，以及超过 50,000 差异体积的“降级模式 (Degraded Mode)”逻辑标准化。移除 `__idbPatchInstalled` 等标志位。

- [ ] **Task 1.3: UI 交互与移动端控制的 AOP 改造**
  - **目标:** 重构 `InventoryUI.prototype._onSlotPointerDown` (拖拽优化) 与 `TouchController.prototype._init` (安全区防误触)。
  - **行动:** 将这些复杂的原生事件绑定拦截逻辑，统一放入 `wrapProto` 提供的安全闭包中，并确保 `PatchManager.results` 能够追踪到 `TouchController` 的覆盖状态。

---

## 阶段二：向事件驱动模型迁移 (Event-Driven Migration)

**策略:** 凡是不需要改变原始方法返回值、不阻断原始执行流程的追加型逻辑，一律从原型链上剥离，改为监听 `window.GameEvents`。

- [ ] **Task 2.1: 剥离冗余的生命周期补丁**
  - **目标:** 移除 `__logicLifecycleInstalled` 和 `__tuGameReadyEvent` 等仅用于初始化的补丁。
  - **行动:** 删除这些修改 `Game.prototype.init` 的代码。原引擎已经在 `init` 末尾抛出了 `game:init:post`。直接使用 `window.GameEvents.on('game:init:post', (game) => { ... })` 来挂载音频安全属性（`tuAudioVisPatch`）和输入重置监听。

- [ ] **Task 2.2: 剥离后处理特效补丁 (Post-Processing)**
  - **目标:** 移除 `Renderer.prototype.applyPostFX` 上的多层拦截（如 `__weatherPostTintInstalled` 和 `__underwaterFogInstalled`）。
  - **行动:** 原引擎在管线末尾触发了 `renderer:postFX:complete`。将雨天色调滤镜和水下噪声雾效的 Canvas 绘制逻辑，直接移入该事件的监听回调中。

- [ ] **Task 2.3: 剥离状态同步补丁**
  - **目标:** 移除对 `SaveSystem.prototype.markTile` 的拦截 (`__logicMarkTilePatched`)。
  - **行动:** 改为监听 `block:written` 事件，在回调中同步触发多线程逻辑引擎的区块更新与版本号自增。

---

## 阶段三：Web Worker 源码注入与内存池重构 (Worker Injection Refactoring)

**策略:** 梳理 `TileLogicEngine._workerSource` 中极其脆弱的正则替换（`.replace`）逻辑，确保其在不同浏览器或引擎版本下的鲁棒性。

- [ ] **Task 3.1: 固化内存复用池 (Buffer Recycling) 注入**
  - **目标:** 确保 `__TU_OUT_POOL__` 和 `ArrayBuffer` 的回收机制（Zero-allocation）正确注入。
  - **行动:** 将分散在多个补丁中的正则替换字符串提取为常数模板。增加注入失败的回退检查：如果正则匹配失败，立即通过 `AppServices.get("logger")` 抛出 `[Worker Inject] Regex Match Failed`，防止 Worker 静默崩溃。

- [ ] **Task 3.2: 固化数据打包压缩 (Data Packing) 注入**
  - **目标:** 优化针对大数据量（>6144 方块更新）时的 `Uint32Array` (坐标) 和 `Uint16Array` (ID) 位运算压缩逻辑。
  - **行动:** 确保 `__TU_PACK_THRESHOLD__` 逻辑被准确包裹在 `TU.PatchRegistry` 的高优先级 (`order`) 阶段注入，防止被后续的其他 Worker 修改补丁覆盖。

---

## 阶段四：全局调度与验证 (Orchestration & Verification)

- [ ] **Task 4.1: 注册表执行流的整理**
  - **行动:** 检查文件末尾的 `TU.PatchRegistry.installAll()` 调用。确保所有的补丁闭包都已替换为 `TU.PatchRegistry.register({ id: "...", order: N, apply: () => {...} })` 格式，彻底消除物理位置带来的加载竞态条件。

- [ ] **Task 4.2: 运行测试与遥测诊断**
  - **行动:** 
    1. 使用 `OpenPreview` 启动服务并加载页面。
    2. 在控制台验证 `window.AppServices.get('patchManagerResults')`，确认所有通过 AOP 注入的补丁状态为 `success`。
    3. 触发天气变化、物品拖拽和放置大量方块，验证事件监听器（EventBus）和 Worker 内存池是否正常运作，且无任何无限递归或 `QuotaExceededError`。

---
**执行确认:**
以上是经过 10 个 Search Agent 并行深度分析后得出的**全面、深入且安全的重构计划**。它不仅解决了表面的代码混乱，更在架构层面将游戏引擎推向了更现代的 AOP 与事件驱动模型。

您是否同意该深度执行计划？请“点头”，我将立即并行开启验证与修改流程。