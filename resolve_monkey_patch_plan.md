# 猴子补丁（Monkey Patch）清理与重构执行计划

目标：把 `/workspace/part3_game_single (6) (9)(9)(2)(6).html` 里当前“动态改原型 / 运行时覆盖方法 / 注入全局状态”的补丁体系，从“分散、不可回收、顺序靠约定”升级为“可追踪、可回滚、可验证、可逐步融合进主逻辑”的结构。本文档仅描述执行计划与验证方法，不包含任何代码改动，也不会产生提交/替换文件等操作；仅在你明确确认（点头）后，才会进入执行阶段并开始修改。

## 1. 现状分析（比“补丁很多”更重要的结构性问题）
该文件是单体 HTML（约 1.5MB），包含大量 IIFE + 运行时改写。当前存在两类“补丁”：

1) 受控补丁：通过 `TU.PatchRegistry.register({id,order,apply})` 先注册，随后在页面尾部统一 `installAll()` 执行 `apply()`（有排序）。  
2) 非受控补丁：不走 PatchRegistry，脚本解析到该段即立刻 `__p.apply()` 或直接覆盖原型/全局对象（没有统一排序/幂等/回滚）。

“乱七八糟”的核心不是补丁数量，而是：补丁改变了“代码的解释权”与“执行顺序”，且缺少可观测性与可逆性，导致任何一个方法的真实行为需要跨多个位置才能还原。

**存在的问题（乱七八糟的原因）：**
1. **受控与非受控混用**：一部分走 PatchRegistry（有 order），另一部分在解析时立即生效（无 order），导致真实执行顺序只能通过“脚本在文件中的位置”推断，维护成本极高。
2. **不可回收（不可逆）**：几乎所有改动都是“install 后永久生效”，没有 `dispose/unpatch`，也没有保存 property descriptor 以便还原。
3. **平台 API 级 monkey patch（风险最高）**：存在对 `HTMLCanvasElement.prototype.getContext` 的覆盖，影响面超出游戏自身代码，且会改变 CanvasContext 上的方法与属性行为。
4. **补丁间互相“压制”而不是组合**：典型例子是 `Renderer.applyPostFX` 被多次包裹，后装补丁通过临时置零等方式禁用前装逻辑，靠隐含约定维持正确效果。
5. **幂等机制不健全**：`PatchManager.once` 先置位后执行，失败后不可重试；且对 async 无效（Promise reject 不会被捕获），容易产生“假成功/漏执行”。
6. **初始化事件存在重复触发风险**：`Game.init` 自身会 emit `game:init:post`，且某补丁又包裹 `init` 后再次 emit，导致所有订阅者潜在二次执行（隐蔽且危险）。
7. **Safe.run 的“吞异常”掩盖问题**：当前 Safe.run 默认吞掉异常并返回 fallback，且对 async 失效，会让关键失败变成“静默功能缺失”，排查困难。
8. **跨线程耦合（Worker）把“补丁状态”变成协议的一部分**：世界生成 Worker 通过序列化主线程上当时的 `WorldGenerator.prototype` 注入 worker，主/worker 的行为一致性依赖“当时装了哪些补丁”。

## 2. 全量盘点（必须覆盖，不允许“漏项导致返工”）
### 2.1 PatchRegistry 受控补丁清单（按 order）
- `batching_idb_pickup_safe_v2` (order=40)：存档与拾取安全性；涉及 IDB/存储回退、chunk canvas 缓存策略、SaveSystem 行为改写。
- `weather_lighting_audio_sync_v1` (order=50)：天气→音频与后处理同步；覆盖 `AudioManager.updateWeatherAmbience`，包裹 `Renderer.applyPostFX`，新增 `TU.forceWeather`。
- `weather_canvas_fx_perf_v1` (order=60)：新增 `TU.WeatherCanvasFX` 与 overlay canvas（雨雪闪电）。
- `tu_experience_optimizations_v3` (order=70)：输入/窗口失焦/可见性恢复、低功耗 CSS；包裹 `Game.init` 并 emit `game:init:post`（需处理与核心 init emit 的重复）。
- `v9_biomes_mines_dynamic_water_pumps_clouds_reverb` (order=80)：大量世界生成/方块表/结构库/音频氛围扩展；再次覆盖 `Renderer.applyPostFX` 并在渲染后绘制 weather overlay；包含对酸雨更新接口的遗留引用（需清理）。
- `tu_weather_rain_visible_fix_v1` (order=90)：覆盖 `WeatherCanvasFX` 的雨雪 pattern 生成与兼容性 ctx 获取（强覆盖）。
- `tu_acid_rain_hazard_v1` (order=100)：新增酸雨机制；覆盖 `WeatherCanvasFX` 多个方法以支持酸雨绘制；新增 UI 闪屏与伤害逻辑；挂 `game:update/game:render:post`。
- `tu_world_worker_patch_v1` (order=110)：世界生成 worker 化 +（可选）渲染解耦预埋；改写 `WorldGenerator.generate` 并包裹多个 Game/SaveSystem 方法同步 tile/light；存在并发 generate 覆盖 `_pendingGen` 的风险。

### 2.2 非 PatchRegistry 的“立即生效补丁/猴补”清单（同样必须纳入治理）
- `experience_optimized_v2`（带 `id/order` 字段但不受 PatchRegistry 管理）：定义后立刻 `__p.apply()`，属于非受控立即生效改写，且其 `order=10` 仅是“自描述字段”，不是实际安装排序依据。
- “Weather system: dynamic weather …” 逻辑块：IIFE 直接 `GameEvents.on("game:update", ...)` 注入天气状态机与 `weatherFx` 写入，不走 PatchRegistry。
- Inventory PointerEvents 拖拽交换增强：IIFE 直接改写库存交互逻辑，不走 PatchRegistry。
- `HTMLCanvasElement.prototype.getContext` 覆盖：平台级 monkey patch，必须列为最高优先级风险点，且需要可回滚/可观测。
- 其它以 `Proto.method = function(){...}` / `Object.defineProperty(...)` 方式直接覆盖且不通过 PatchRegistry 的逻辑块：阶段一会统一纳入“补丁台账”，禁止继续出现“无登记即改写”。

## 3. 解决目标（可验证的目标，不是口号）
- **可追踪**：任意一个被改写的方法，都能在 1 处查到“谁改的、改成什么、安装顺序、是否成功、是否可回滚”。
- **可回滚**：每一个补丁都具备 `dispose()` 能力（至少恢复原方法引用与 property descriptor）。
- **可验证**：有一份“补丁一致性/重复执行/漏执行/异常收敛”的检查清单，并能在页面内输出结果。
- **可组合**：同一方法的多个需求通过“显式扩展点（pipeline/hook）”组合，而不是链式 wrapper 互相压制。
- **可演进**：允许先“工程化补丁体系”（风险最低），再“逐步融合到主逻辑”（治本），最终可以选择是否完全移除补丁引擎。

## 4. 方案评估（按“风险/收益/可回滚性”排序）
- **方案 A：补丁工程化（推荐先做，最低风险）**
  - 目标：不改变功能，仅把“补丁变更”变成可追踪/可回滚/可验证。
  - 关键动作：建立补丁台账；统一幂等 key；给每个 patch 返回 `dispose()`；新增 `runAsync` 收敛 async 异常；修复 `game:init:post` 重复 emit。
- **方案 B：渐进式融合（推荐主线，治本）**
  - 目标：逐步减少“改原型”的必要性，把逻辑迁到明确的系统边界（WeatherSystem/WorldGen/RendererPipeline/InputLifecycle）。
  - 关键动作：把“强覆盖点”变成“显式 pipeline”；把“副作用型补丁”迁到事件/系统 update；把 Worker 协议与实现显式版本化。
- **方案 C：全面模块化工程化（可选，高成本）**
  - 目标：拆单体 HTML 为 ES Modules 项目（构建/调试/测试体系更完整）。
  - 说明：这不是本次必须项，除非你明确要上工程化构建链路。

## 5. 推荐执行步骤（分阶段、每阶段都有回滚点与验收条件）
推荐路线：先 A 后 B。原因：先让系统“可控”，再做结构性改动；否则任何融合都缺少回滚与观测，风险不可接受。

### 阶段零：基线与开关矩阵（强烈建议先补齐，否则后续无法验收/回滚）
验收：同一版本内可在不改代码的情况下切换 baseline/新实现，并能输出对照数据。
1. 定义开关矩阵（全局开关 + 子系统开关 + 作用域开关）：至少覆盖 Renderer 后处理、Weather overlay、Worker 世界生成、Canvas getContext。
2. 定义基线口径：存档读写成功率、世界生成成功率、异常（同步/异步）计数、以及帧耗时（p50/p95/p99）的采集方式与窗口。
3. 定义回滚门禁：每阶段的“必须回滚”的阈值（例如错误率升高、p99 长帧显著上升、存档失败出现新增类型）。

### 阶段一：建立“补丁台账”与真实执行流（目标是不改功能）
验收：能输出一份“当前页面加载后，哪些补丁成功/失败/跳过、安装顺序、耗时、覆盖了哪些方法”的报告。
1. 盘点所有补丁点（含非 PatchRegistry）并生成台账：id、类型（受控/非受控）、安装时机、改写对象、依赖项、风险等级。
2. 统一幂等策略：以 `patch:${id}` 为唯一 key，禁止补丁内部再用另一套 key；修订 `PatchManager.once` 的失败可重试与 async 语义（至少不再“假成功”）。
3. 修复初始化事件的重复 emit：明确 `game:init:post` 只触发一次，并为需要“二阶段挂载”的逻辑提供单独的 phase（如 `game:init:afterPatches`）。
4. Safe.run 并轨错误收敛：增加 `runAsync` 并接入统一错误上报/面板（不改变原控制流，但保证可见）。

### 阶段二：把“强覆盖点”改造成显式扩展点（目标是组合而不是互相压制）
验收：同一热点方法不再被多次链式 wrapper 覆盖，而是通过 pipeline 顺序执行。
1. Renderer 后处理：把 `applyPostFX` 收敛为单一管线实现，并把现存 pass 明确列出来（至少包含 base postFX、weather tint/lightning、underwater fog），通过 guard 统一处理 `postFxMode/reducedMotion` 等开关；移除“后装补丁用置零压制前装逻辑”的做法。
2. overlay 语义决策：明确 WeatherCanvasFX overlay 是否“吃后处理”。若要保持现状，应固定为“postFX 之后绘制”的一条路径，并与 damage flash/屏幕冲击 overlay 的顺序统一。
3. 天气系统：以 `game.weather` 为真源，集中生成 `weatherFx`，不允许多个补丁在不同地方分别改写 `weatherFx.post*`。
4. WeatherCanvasFX：把 “rain_visible_fix/acid_rain” 从“覆写 prototype 方法”改为“显式参数选择 pattern”，避免后装覆盖前装；并明确 acid 的视觉组成（绿雨 pattern / 闪电 / 是否额外 tint）。

### 阶段三：逐个系统融合（开始“治本”，每次只动一条链）
验收：每完成一个系统，就能删除对应补丁的“原型覆盖部分”，且功能一致。
1. WorldWorker：解决并发 generate 覆盖 `_pendingGen`、消息乱序与状态漂移；补齐协议版本与实现 hash；明确 reject/queue/cancel 策略；失败回退需可观测且能复位；beforeunload/healthcheck 必须 reject pending，避免挂起。
2. v9 世界生成/方块表扩展：把世界生成关键函数从“patch 注入 prototype”迁到 `WorldGenerator` 源码路径（或显式 hook），并补齐测试与可视化验证点。
3. 平台级 Canvas getContext 覆盖：改为局部包装（仅游戏 canvas），并提供可回滚；禁止污染所有 canvas。

### 阶段四：收尾与去补丁引擎（可选，取决于你是否还需要热补丁能力）
验收：补丁系统要么被彻底移除，要么退化为“少量、可回滚、可观测”的热修复机制。
1. 只要仍存在“线上热修复诉求”，就保留 PatchRegistry，但强制：可回滚、依赖声明、phase、runAsync、统一错误收敛。
2. 若确认不再需要热补丁：移除 PatchRegistry/safeApplyPatch/once 等基础设施，并把所有逻辑迁回模块化系统。

## 6. 风险控制（必须提前写死，否则执行时会被动）
- 每次只动一个系统链路：Renderer 后处理 / 天气 / Worker / 世界生成 / Canvas 平台 patch，禁止“同时动多处”。
- 每个阶段必须有回滚点：保留旧实现入口（或保留 patch 并可一键停用），并能在同一版本内切回。
- 对外部行为敏感点（存档、世界生成、输入、音频）必须先建立“行为基线”再改。

## 7. 验证清单（仅用于后续执行阶段；未确认前不进行任何改动与验收）
- 补丁安装报告：所有补丁成功/失败/跳过统计一致，且与台账一致。
- `game:init:post` 仅触发一次（订阅者不会重复执行）。
- Safe.run 同步与异步错误都能进入统一错误面板（不依赖 toast）。
- 天气：clear/rain/snow/thunder/bloodmoon/acid 的视觉与音频均可触发且可关闭；`applyPostFX` 只存在一条管线实现。
- Worker：支持/不支持 Worker 都能生成世界；失败回退可靠；并发触发不会挂起；beforeunload 能清理资源。
- 性能（必须可复现对照，给出原始数据与脚本）：
  1) Frame Time（渲染帧）：固定场景跑 60s，统计 rAF 间隔的 p50/p95/p99 与长帧（>50ms）次数；目标：p99 不回退或回退 < 5%。
  2) Tick Time（主循环）：对 `Game.update` / `Game.render` / `Renderer.applyPostFX` 做埋点，输出每段耗时分布（p95/p99）与调用次数，确保无新增尖峰。
  3) Wrapper/管线拓扑：启动后输出热点函数的最终拓扑（wrap 层数或 pipeline 节点列表）；目标：热点函数不再出现多层链式 wrapper（上限明确且可检验）。
  4) 稳定性：统计 console error + 未捕获异常 + Promise rejection 的计数/分钟；目标：不高于基线。

---

请审阅本文档。如果你认可这份增强版执行计划，我再进入“执行阶段”。进入后我会先提交一份“拟修改点清单 + 风险/回滚方案 + 验收口径”供你二次确认；你再次确认后，才开始修改 HTML 并按阶段验证。你也可以指定只做方案 A（工程化补丁）或只优先处理某个高风险点（例如 Canvas getContext 覆盖或 init 重复触发）。
