# Phase 0 Report: Safety Net & Performance Baselines

## Date: 2026-04-20
## Status: COMPLETED
## Backup: `part0_phase0_safety_net.html`

---

## What Was Done

### 1. TU.Safe.run - Enhanced Error Reporting
- **Before**: `run()` silently swallowed errors by default (returned `undefined`/`fallback`). Only rethrew if explicitly told to.
- **After**: `run()` now ALWAYS reports errors to `TU_Defensive.ErrorReporter` in addition to logging. Errors are visible in the error dialog UI.
- **New**: Added `runAsync()` method for proper async/Promise error handling with retry support.

### 2. GlobalErrorBoundary - New Subsystem
- Added `TU.GlobalErrorBoundary` with `guard()` and `guardAsync()` methods
- Provides explicit error surfacing for all subsystems (Worker, IDB, AudioManager, etc.)
- Supports retry logic for transient failures
- Always logs to `console.error` (never silent)
- Reports to ErrorReporter for UI visibility

### 3. Fixed Double `game:init:post` Emission
- **Before**: `game:init:post` fired TWICE:
  1. Original `Game.prototype.init()` at end of init (line ~23913)
  2. Patch wrapper in "experience optimization v3" re-emitted it (line ~28889)
- **After**: Removed the duplicate emission from the patch wrapper
- **Impact**: Eliminates double initialization of TileLogicEngine, double machine indexing, and first-frame stutter spike

### 4. PerformanceBaseline - Frame/Tick Tracking
- Added `TU.PerformanceBaseline` with p50/p95/p99 metrics for both frame time and tick time
- Auto-starts when game RAF loop begins
- Auto-prints baseline report after 30 seconds
- Hooked into `game:update:pre/post` and `game:render:pre/post` events
- Available via `TU.PerformanceBaseline.printReport()` in console

### 5. PatchAuditLedger - Patch Cataloging
- Added `TU.PatchAuditLedger` for registering and tracking all monkey patches
- Methods: `register()`, `getAll()`, `getByTarget()`, `printSummary()`
- Foundation for tracking which patches have been absorbed in later phases

### 6. Storage Error Surfacing
- **localStorage**: `QuotaExceededError` now reported as `console.error` + ErrorReporter + toast (was silently swallowed)
- **IndexedDB**: `get()` and `set()` failures now reported as `console.error` + ErrorReporter + toast (were silent `console.warn`)

---

## Patch Inventory (Pre-refactor Audit)

| Flag | Target | Method | Count |
|------|--------|--------|-------|
| `__tuWorldApiInstalled` | `world` | `getTile` | 1 |
| `__idbPatchInstalled` | `SaveSystem` | `clear/save/prompt` | 1 |
| `__chunkBatchSafeInstalled` | `Renderer.prototype` | chunk batching | 1 |
| `__tileLogicInstalled` | `TU` | TileLogicEngine | 1 |
| `__logicLifecycleInstalled` | `Game.prototype` | lifecycle hooks | 1 |
| `__rainSynthInstalled` | `AudioManager.prototype` | rain synthesis | 1 |
| `__weatherPostTintInstalled` | `Renderer.prototype` | `applyPostFX` wrap | 1 |
| `__weatherCanvasFxRenderInstalled` | `Game.prototype` | weather render | 1 |
| `__tuGameReadyEvent` | `Game.prototype` | `init` wrap (FIXED) | 1 |
| `__tuInputSafety` | `InputManager.prototype` | `bind` wrap | 1 |
| `__chestLootInstalled` | `Game.prototype` | chest loot | 1 |
| `__workerInstalled` | `TileLogicEngine` | worker source | 1 |
| `__underwaterFogInstalled` | `Renderer.prototype` | `applyPostFX` wrap | 1 |
| `__cloudBiomeSkyInstalled` | `Renderer.prototype` | sky rendering | 1 |
| `__machinesInstalled` | `Game.prototype` | machine logic | 1 |
| `__caveReverbInstalled` | `AudioManager.prototype` | cave reverb | 1 |
| `__tuPoolInstalled` | `Toast` | pooling | 1 |

### applyPostFX Onion Layers (4 wraps deep):
1. **Original**: `Renderer.prototype.applyPostFX` (base class, line ~20731)
2. **Weather Post Tint**: `__weatherPostTintInstalled` wraps #1
3. **Weather Optimized**: `__weatherPostTintOptimized` wraps #2
4. **Underwater Fog**: `__underwaterFogInstalled` wraps #3

---

## Risk Assessment
- All changes are additive/non-breaking
- No rendering logic changed
- No game state logic changed
- Error reporting is always wrapped in try/catch to prevent cascading failures

## Next Phase
Phase 1: Native Class Integration & Timing Fixes - Merge InputManager, AudioManager, TouchController patches into base classes.
