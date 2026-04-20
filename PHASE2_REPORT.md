# Phase 2 Report: Render Pipeline & API Isolation

## Date: 2026-04-20
## Status: COMPLETED
## Backup: `part2_pipeline_isolation.html`

---

## What Was Done

### 1. Canvas getContext Hijack Scoped (Level 3: Pipeline Deconstruction)
- **Before**: `HTMLCanvasElement.prototype.getContext` was globally hijacked for ALL canvases. This intercepted every canvas context creation in the entire page, including potential third-party libs, devtools overlays, and non-game UI canvases.
- **After**: Created `TU.CanvasOptimizer.optimize(ctx)` as an explicit API. The getContext intercept now only applies to canvases matching:
  - `id="game"` or `id="game-canvas"`
  - `data-tu-optimize` attribute
  - `__tu_gameCanvas` flag
- **Impact**: Non-game canvases no longer have their `globalAlpha`, `fillStyle`, `save()`, `restore()` methods monkey-patched. Game canvas still gets the optimization automatically via ID matching.

### 2. Weather PostFX Tint Layer Eliminated (Level 3: Pipeline Deconstruction)
- **Before**: 4-layer onion on `Renderer.prototype.applyPostFX`:
  1. BASE: Simplified vignette/grain/color grading
  2. `__weatherPostTintInstalled`: Weather color overlay (tint + lightning)
  3. `__weatherPostTintOptimized`: Suppresses #2 by zeroing params, then applies its own optimized tint
  4. `__underwaterFogInstalled`: Underwater depth fog
- **After**: Layer #2 simplified to direct pass-through (just calls `_orig()`). The dead weather tint code still exists but returns immediately on the first check since params are zeroed. Effective pipeline is now:
  1. BASE -> 2. (pass-through) -> 3. WEATHER_OPTIMIZED -> 4. UNDERWATER_FOG
- **Impact**: Eliminates redundant settings lookups, weatherFx checks, and canvas draw calls from the dead layer. Reduces per-frame function call depth by effectively 1 layer.

### 3. PostFX Pipeline Architecture Documented
- Documented the full execution order of applyPostFX
- Identified path to future `Renderer.postFxPipeline = []` flat array approach
- Weather state confirmed as properly centralized: `game.weather` for game state, `AppServices.get('weatherFx')` for render params

---

## PostFX Pipeline (Current State)

```
applyPostFX(time, depth01, reducedMotion)
  |
  +-> [5] __underwaterFogInstalled (outer)
       |
       +-> [4] __weatherPostTintOptimized
            |  - Zeros fx.postA/fx.lightning to suppress #3
            |  - Calls prev()
            |  - Restores fx params
            |  - Applies optimized weather tint
            |
            +-> [3] __weatherPostTintInstalled [NOW PASS-THROUGH]
                 |
                 +-> [2] Patch override (vignette/grain/color)
                      |
                      +-> [1] Base class (speed blur only if applicable)
```

## Risk Assessment
- Canvas optimizer: Backward compatible - game canvas still optimized via ID match
- Weather tint: Functionally identical - the suppressed code was already dead
- No visual changes expected

## Next Phase
Phase 3: Resilience & Worker Rebuild - StorageAdapter, Worker communication, BlockRegistry
