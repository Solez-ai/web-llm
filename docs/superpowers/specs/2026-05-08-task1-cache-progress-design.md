# Task 1 (Issue #710): Cache-Load Progress Callbacks — Design

## Problem

`initProgressCallback` currently provides meaningful progress when model artifacts are downloaded from the network, but provides little-to-no feedback when artifacts are loaded from browser cache (CacheStorage / IndexedDB / OPFS). Warm-start loads can still take multiple seconds (notably when deserializing tensor cache), and users see “0%” progress despite active work.

Issue: https://github.com/mlc-ai/web-llm/issues/710

## Goals

- Preserve backward compatibility for existing `initProgressCallback` usage.
- Provide a way for UIs to distinguish which loading phase is happening (`download` vs `cache-load` vs `initialize`).
- Improve cache-load progress by computing a non-zero percentage when underlying runtime emits shard progress in `report.text` (e.g. `[loaded/total]`).
- Provide minimal but meaningful stage messages even when byte/shard totals are unavailable.

## Non-goals

- Deep per-shard byte accounting by directly reading CacheStorage entries in WebLLM TypeScript. This depends on internal cache formats in `@mlc-ai/web-runtime` and is higher risk.
- Changes to `@mlc-ai/web-runtime` itself.

## API Surface Changes

- Extend `InitProgressReport` with optional fields:
  - `stage?: "download" | "cache-load" | "initialize"`
  - `current?: number`
  - `total?: number`
- Keep existing fields (`progress`, `timeElapsed`, `text`) unchanged.

## Implementation Plan

### 1) Normalize runtime progress reports

Add a small helper (exported from `src/support.ts`) that:

- Infers `stage` from `report.text`:
  - contains `"cache"` → `stage = "cache-load"`
  - contains `"download"` (or `"fetch"`) → `stage = "download"`
  - otherwise → `stage = "initialize"`
- Extracts progress counts from text using a conservative pattern like `[...]`:
  - Example: `"Loading model from cache [3/12]"` → `current=3`, `total=12`
- If the inferred stage is `cache-load` and `report.progress` is `0`, replace `progress` with `current / total` (clamped to `[0,1]`) when available.

### 2) Use the normalizer everywhere we register init progress callbacks

In `src/engine.ts`, when registering the init progress callback with the TVM runtime (`tvm.registerInitProgressCallback`), wrap the user callback with the normalizer so downstream consumers see:

- consistent `stage` values
- non-zero progress for cache loads when shard counts are present in `text`

Also update the final “Finish loading…” report emitted by WebLLM to include `stage: "initialize"`.

In `src/service_worker.ts`, ensure the “Already loaded the model. Skip loading” progress event includes `stage: "initialize"` as well.

### 3) Example update

Update an existing example (likely `examples/cache-usage/`) to render:

- `report.stage` (if present)
- a percentage from `report.progress`
- `report.text`

### 4) Tests

Add unit tests (Jest) for the normalizer helper:

- Cache-load text with shard counts adjusts progress from `0` to `current/total` and sets `stage`.
- Download text sets `stage="download"` and does not override non-zero progress.
- Text without pattern yields `stage="initialize"` and keeps original progress.

## Validation

- `npm test` passes (new unit tests added).
- Run the updated example and verify that warm starts show incremental progress (or at least a correct stage) rather than a persistent 0%.

## Follow-ups

- If `@mlc-ai/web-runtime` adds first-class cache-load byte/shard progress in `report.progress`, the normalizer can become a no-op for that scenario while keeping `stage` semantics useful to UIs.

