# Phase 3 Report: Resilience & Worker Rebuild

## Date: 2026-04-20
## Status: COMPLETED
## Backup: `part3_resilience_worker.html`

---

## What Was Done

### 1. StorageAdapter - Unified Degradation Flow (Level 4: Pluginification)
- Created `TU.StorageAdapter` class with explicit LS -> IDB -> None degradation
- **write(key, value)**: Tries localStorage, falls back to IDB on QuotaExceededError
- **read(key)**: Reads from both LS and IDB, returns freshest based on timestamp
- **getStatus()**: Reports current tier, quota status, IDB health
- All failures explicitly logged and reported to ErrorReporter
- Registered as `AppServices.get('storageAdapter')`

### 2. BlockRegistry - Stable ID Mapping with Palette Support (Level 4: Data Baking)
- Created `TU.BlockRegistry` with canonical name <-> ID mapping
- **init()**: Builds registry from frozen `BLOCK` constant
- **exportPalette()**: Generates `{ version, palette: { name: id } }` for save headers
- **validatePalette(savedPalette)**: Validates saved palette against current registry, returns remap table if IDs changed
- Future-proofs against ID drift: saves can embed their palette, and the registry can remap on load
- Registered as `AppServices.get('blockRegistry')`

### 3. Worker beforeunload Cleanup (Level 4: Worker Rebuild)
- **Before**: Only terminated WorldWorkerClient worker on unload, didn't clean up pending Promises
- **After**: 
  - Rejects pending `_pendingGen` Promise before terminating (prevents zombie Promise hangs)
  - Clears `_frameInFlight` flag
  - Nullifies worker reference to prevent post-terminate messaging
  - Also terminates TileLogicEngine worker (was missed before)
  - Closes and nullifies `_lastBitmap` to prevent memory leak

---

## Architecture Additions

### StorageAdapter Degradation Flow
```
write(key, value)
  |
  +-> Try localStorage
  |     |
  |     +-> Success -> return { ok: true, tier: 'ls' }
  |     |
  |     +-> QuotaExceededError -> mark _quotaExceeded
  |
  +-> Try IndexedDB
  |     |
  |     +-> Success -> return { ok: true, tier: 'idb' }
  |     |
  |     +-> Failure -> mark _idbFailed, report error
  |
  +-> return { ok: false, tier: 'none' }
```

### BlockRegistry Palette Mapping
```
Save Header:
{
  version: 1,
  palette: {
    AIR: 0, DIRT: 1, GRASS: 2, STONE: 3, ...
  },
  diff: [...]
}

On Load:
  1. Extract palette from save header
  2. BlockRegistry.validatePalette(savedPalette)
  3. If remap needed, apply remap to diff entries
  4. Load world with current IDs
```

---

## Block ID Analysis
- BLOCK is `Object.freeze()`'d - no runtime modification possible
- ~140+ block types with static numeric IDs (0-140+)
- BLOCK_DATA is also static - no dynamic block type addition
- Current risk: LOW (no runtime ID drift)
- Future risk: MEDIUM (if mods or updates add blocks at different IDs)
- BlockRegistry palette mapping provides forward compatibility

## Risk Assessment
- StorageAdapter: Purely additive, doesn't modify existing save flow
- BlockRegistry: Read-only analysis of BLOCK constant, no mutation
- beforeunload: More thorough cleanup, prevents Promise hang and memory leak

## Next Phase
Phase 4: Cleanup & Final Validation
