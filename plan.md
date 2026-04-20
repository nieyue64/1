# 猴子补丁 (Monkey Patch) 深度治理与重构计划 (v2)

通过对代码库进行 10 个维度的并发深度扫描，我们发现当前的“猴子补丁”问题远比预期的严重。系统不仅存在逻辑碎片的表象问题，更在内存管理、多线程通信、异常处理和数据持久化层面埋下了大量“定时炸弹”。

## 1. 核心痛点深度剖析 (Root Cause Analysis)

### 1.1 高危的多线程代码注入 (Web Worker Stringification)
- **现状**：为了将 `WorldGenerator` 放入 Worker 避免主线程阻塞，补丁通过 `fn.toString()` 提取主线程类方法，拼接成字符串后注入 Blob 运行。
- **致命缺陷**：
  1. **闭包上下文完全丢失**：导致大量局部变量和外部依赖在 Worker 中引发 `ReferenceError`，不得不靠手动塞入字符串（如伪造环境）来修补。
  2. **工程化死局**：极度不兼容 Vite/Webpack 和代码混淆（Minification），一旦变量名被压缩（如变成 `a`, `b`），基于字符串的方法名查找将瞬间崩溃。

### 1.2 极度脆弱的原型链与全局状态污染 (Prototype & Global Pollution)
- **现状**：为了防止重复打补丁，系统在全局 `window` 和各个类的 `prototype` 上挂载了超过 40 个“补丁锁”（如 `__chunkBatchSafeInstalled`、`window.__TU_DEBUG__`）。
- **致命缺陷**：
  1. **性能劣化**：高频执行的方法（如 `_writeTileFast`, `applyPostFX`）被反复包装，破坏了 JS 引擎的 JIT 优化和内联缓存（Inline Caching）。
  2. **内存泄漏**：如 `window.__activeTimers` 这种全局 Set 未严格清理，极易造成内存溢出。
  3. **调用栈灾难**：错误堆栈充满匿名闭包，问题定位极难。

### 1.3 异常的防御性吞噬 (Aggressive Error Swallowing)
- **现状**：补丁系统中充斥着 `try { ... } catch(e){ console.warn(e); }`。
- **致命缺陷**：本该是**致命错误（Fatal Errors）**的问题被降级成了**静默逻辑缺陷（Silent Bugs）**。
  1. **幽灵方块**：方块 ID 越界导致底层 `SOLID` 数组写入失败被吞噬，玩家穿模。
  2. **僵尸存档**：清除存档时 IndexedDB 删除报错被吞噬，导致下次依然加载旧存档。
  3. **竞态条件**：音频节点的 `InvalidStateError` 被掩盖，导致多个系统争夺同一个音频上下文。

### 1.4 数据字典的硬编码与 ID 漂移 (Hardcoding & ID Drifting)
- **现状**：方块 ID 分配使用 `allocId` 盲猜（遍历找空位），生物群系生成逻辑被硬编码为坐标比例（`t < 0.34 ? "forest" : "desert"`）。
- **致命缺陷**：方块 ID 强依赖补丁加载顺序。新增或删除补丁会导致 ID 漂移，进而引发**旧存档数据大面积损坏（方块错乱）**。完全摧毁了第三方 Mod 扩展的可能性。

### 1.5 逻辑严重碎片化与管线冲突 (Logic Fragmentation)
- **现状**：一个完整的“天气系统”被割裂在 5 个独立的补丁中；渲染器的后处理（PostFX）被酸雨、雷声、底层水雾等多层拦截。
- **致命缺陷**：补丁间存在隐式的“补丁盖补丁”冲突。例如雨滴渲染补丁甚至需要专门写回退代码来修复前一个性能优化补丁带来的 Canvas 上下文丢失问题。

---

## 2. 全面重构战略与执行路径

为彻底根除技术债务，我们将采取“解耦、静态化、事件驱动、注册表化”四大战略，分五个阶段进行平滑重构。

### 阶段一：工程化基建与多线程剥离 (Build System & Worker Decoupling)
*解决：代码压缩不兼容、闭包丢失、单文件臃肿*
1. **引入 Vite/Webpack**：将 1.5MB 的单文件拆分为标准的 ES Modules (`.ts`/`.js`)。
2. **废弃字符串注入**：彻底删除 `fn.toString()` 注入逻辑。将 `WorldGenerator` 和离屏渲染（OffscreenCanvas）抽离为物理上独立的 `worker.js`。
3. **标准通信机制**：主线程与 Worker 之间建立基于 `postMessage` 和 `SharedArrayBuffer` 的标准化通信协议。

### 阶段二：生命周期重构与事件驱动 (Event-Driven Architecture)
*解决：原型链污染、逻辑碎片化、调用栈过深*
1. **引入 EventBus**：构建全局类型安全的事件总线。
2. **剔除原型链劫持**：将 `Class.prototype.method = ...` 的拦截模式，替换为核心循环中的标准生命周期钩子（如 `emit('onWeatherUpdate')`，`emit('beforeRender')`）。
3. **聚合碎片逻辑**：将散落在 5 个补丁中的天气逻辑，聚合为一个高内聚的 `WeatherSystem` 模块。

### 阶段三：注册表模式与数据烘焙 (Registry Pattern & Data Baking)
*解决：ID漂移、存档损坏、扩展性差*
1. **建立标准注册表**：实现 `BlockRegistry` 和 `BiomeRegistry`。所有扩展内容必须通过带有命名空间的字符串 ID（如 `dlc:acid_pump`）进行注册。
2. **编译期数据烘焙 (Data Baking)**：为了保持底层 `TypedArray`（如 `SOLID[id]`）的 $O(1)$ 极致性能，在游戏 `PostInit` 阶段，将注册表中的面向对象数据“扁平化烘焙”回全局数组中。
3. **存档 ID 映射表 (Palette Mapping)**：在存档头部引入方块 ID 映射表，解决历史数字 ID 到新字符串 ID 的兼容降级问题，杜绝“ID漂移”导致的坏档。

### 阶段四：渲染管线与输入层重写 (Pipeline & Input Normalization)
*解决：后处理性能雪崩、系统级交互劫持*
1. **标准化渲染管线**：移除对 `Renderer.applyPostFX` 的多层闭包劫持。实现 `PostProcessPipeline`，强制为每个 Effect 节点分配独立的 `try-catch` 和 `ctx.save/restore` 沙盒，彻底消除 Canvas 状态污染丢失风险。
2. **消除 Overdraw (过度绘制)**：将各后处理节点（酸雨、暗角、水雾）从“直接执行全屏 `fillRect`”重构为“参数预混合 (Parameter Baking)”模式。由调度器在内存中叠加所有激活效果的颜色与混合模式参数，最终仅执行单次全屏绘制，从物理层面消除性能雪崩。
3. **重构触摸防劫持**：将 `TouchController` 中的动态安全区（Safe Zones）计算规范化，统一通过 UI 层的 Pointer Events 拦截，而非暴力 `preventDefault()` 画布事件。

### 阶段五：异常处理治理与容灾降级 (Error Handling & Storage Fallback)
*解决：静默逻辑缺陷、存储失败*
1. **拆除无声炸弹 (Fail-Fast)**：移除所有吞噬异常的全局 `try-catch`。引入统一的 `GlobalErrorBoundary`，将“幽灵方块”（底层数组越界）和“竞态条件”（音频节点抢占）等致命错误直接暴露并阻断，拒绝带病运行。
2. **规范化存储降级**：将 `SaveSystem` 中的 `LS -> IDB -> Lite 模式` 降级逻辑从补丁拦截转正为核心原生流程。当 IndexedDB 真正损坏或超限时，通过显式异常捕获触发降级路由，阻断虚假的 UI 成功提示，彻底解决“僵尸存档”问题。

---
**执行准则**：重构将遵循“逐步替换，双轨并行”的原则。每完成一个子系统的模块化（如先重构 `InputManager`），立即进行单元测试与人工验证，确保不影响主流程的稳定性。同时，在 Worker 通信改造时，需保留 `Transferable Objects` 机制作为 `SharedArrayBuffer` 在无 COOP/COEP 跨域隔离环境下的容灾降级方案。