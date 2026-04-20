# 猴子补丁（Monkey Patch）深度清理与重构执行计划

**目标:** 解决 `part3_game_single (6) (9)(9)(2)(6).html` 中因 9 个高耦合度的运行时补丁引发的“俄罗斯套娃”式包裹、原型链污染、Web Worker 序列化割裂以及潜在的性能抖动问题。通过将补丁逻辑静态化合并至核心类，彻底消除猴子补丁架构。

---

## 1. 现状深度诊断 (Deep Analysis)

通过对代码库的并发深度检索及 10 个 Agent 的交叉验证，我们发现当前的补丁系统存在以下严重的技术债务：

1. **“俄罗斯套娃”式的方法劫持 (Russian Doll Wrapping)**:
   - `Renderer.prototype.applyPostFX` 被连续劫持了 **4次**（Patch 10 -> 50 -> 60 -> 80）。
   - `Game.prototype._writeTileFast` 和 `SaveSystem.prototype.applyToWorld` 均被劫持 **2次**。
2. **Web Worker 上下文割裂**:
   - Patch 80 为 `WorldGenerator` 注入了大量新生态和多层矿洞逻辑。
   - Patch 110 将世界生成移入 Web Worker，被迫使用 `.toString()` 配合正则 `_fnToExpr` 将主线程函数转换为字符串注入 Worker，极度脆弱。
3. **系统级逻辑外挂**:
   - `tu_experience_optimizations_v3` (Patch 70) 强行外挂了输入安全网（防卡键）、滚轮切换快捷栏以及低功耗模式。
   - 音频系统（混响、雷雨白噪声合成）、动态水泵、掉落物拾取批处理（Patch 40）等均游离在核心类之外。
4. **全局污染与副作用**:
   - 注入了大量 `<style>` 和 DOM（`#damage-flash`, `#tu-profiler`）。
   - 挂载了数十个全局变量（`window.__TU_PERF__`, `window.TU`）。

---

## 2. 核心重构策略 (Refactoring Strategy)

我们将按类（Class）而非按补丁（Patch）来进行重构，将分散的逻辑“压平（Flatten）”并合并到原始类定义中。

### 阶段一：核心游戏逻辑解耦与合并
- [ ] **重构 `Game` 类**:
  - `_writeTileFast`: 整合一维数组更新、机械/水流版本号管理，以及 Web Worker 同步逻辑。
  - `_updateLight`: 整合光照多线程同步逻辑。
  - `_startRaf` & `_rafCb`: 原生整合 `__TU_PERF__` 性能监控，并**清理或原生集成 `#tu-profiler` UI**。
  - 补充原生注册：动态水泵（`PUMP_IN/OUT`）、压力板逻辑及掉落物（`DroppedItemManager`）拾取动画批处理。
- [ ] **重构 `WorldGenerator` (解决 Worker 序列化痛点)**:
  - 在原始类中原生写入 `_biome`、`_getSurfaceBlock`、`_generateMultiLayerMines` 等逻辑。
  - 重写 `generate`，原生支持异步 Web Worker 委派，废弃 `.toString()` 函数字符串化注入。

### 阶段二：渲染管线与特效压平
- [ ] **重构 `Renderer` (拆解套娃)**:
  - 压平 `applyPostFX`: 将基础噪点、天气色调覆盖、水下雾气合并为单一函数，消除 `_orig.call` 开销。
  - 压平 `__cb2_getEntry`: 将 LRU 缓存、Canvas 对象池及修复边缘截断的 `glowCanvas` 双图层逻辑直接写入原始定义。
- [ ] **重构 `WeatherCanvasFX`**:
  - 整合 `_ensure2d` 兼容性后备方案及离屏 Pattern 渲染。
  - 原生植入酸雨逻辑：绿色雨滴、环境光滤镜、遮挡判定与屏幕伤害闪烁（DOM `#damage-flash`）。
  - 补充云朵（`_clouds`）独立渲染逻辑。

### 阶段三：持久化与系统基建
- [ ] **重构 `SaveSystem`**:
  - 原生集成 IndexedDB（`tu_terraria_ultra_save_db_v1`）。
  - 写入“降级模式”，支持大存档的 `KEY_FULL` 与 `KEY_LITE` 拆分。
  - 更新 `applyToWorld`：包含 RLE 差异解码与 Worker 渲染同步。
- [ ] **清理补丁基础设施**:
  - 彻底删除 `TU.PatchRegistry` 和 `PatchManager` 类定义及所有注册代码。
  - 移除防重包裹标志位 `__tu_wrap_*` 和 `window.__TU_PATCH_FLAGS__`。

### 阶段四：输入输出与交互体验 (I/O & UX)
- [ ] **重构 `InputManager` / `TouchController`**:
  - 原生集成鼠标滚轮切换快捷栏逻辑。
  - 原生集成页面失焦（`blur`, `visibilitychange`）的防卡键安全网。
  - 整合移动端摇杆/按钮的安全区（Safe Zone）防误触计算。
- [ ] **重构 `AudioManager`**:
  - 原生集成天气雨声合成（`_makeLoopNoiseBuffer`）、雷暴视听同步以及洞穴混响效果。
  - 整合页面后台挂起/恢复的 WebAudio 节能逻辑。

---

## 3. 测试与验证标准

- **验证渲染**: 天气（酸雨/闪电）、水下雾气、发光方块边缘（Glow Padding）无裁切。
- **验证生成**: 新世界必须包含沙漠、雪地、神庙结构及多层矿洞。
- **验证 Worker**: WorldWorkerClient 正常运作，无需 `.toString()` 传递函数。
- **验证存档**: 大存档成功写入 IndexedDB，读取时同步给渲染 Worker。
- **验证交互**: 滚轮切栏顺畅，切后台不卡键，移动端无误触，音频雷声与闪电画面严格同步。