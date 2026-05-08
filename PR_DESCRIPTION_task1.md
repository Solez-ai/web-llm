# [Task1] Cache-load progress callbacks (Issue #710)

## Problem

`initProgressCallback` provides useful progress during network download, but gives near-zero feedback on warm starts when model weights are already cached. Loading from cache can still take multiple seconds, and UIs often show “0%” throughout.

Issue: https://github.com/mlc-ai/web-llm/issues/710

## What changed

- Extended `InitProgressReport` with optional fields:
  - `stage?: "download" | "cache-load" | "initialize"`
  - `current?: number`
  - `total?: number`
- Added a small normalizer (`normalizeInitProgressReport`) that:
  - infers `stage` from `report.text`
  - extracts shard counts from `report.text` when present (e.g. `[3/12]`)
  - converts cache-load “0%” reports into a computed percentage (`current / total`) when possible
- Wrapped the TVM init progress callback registration in `MLCEngine` so all progress events go through the normalizer.
- Updated the service worker “already loaded” fast-path to emit `stage: "initialize"`.
- Updated `examples/cache-usage` to display `stage` and percent.

## How to test

```bash
npm ci
npm test
npm run build
npm run lint
```

Manual (browser) sanity check:

- Run `examples/cache-usage/` and load a model once (cold start), then reload the model (warm start).
- On warm start, the UI should show `stage` and a non-zero percent when shard counts are present in the progress text.

## Backward compatibility

- No existing callback signatures changed.
- New fields are optional; existing code reading `{ progress, timeElapsed, text }` continues to work unchanged.

## Known limitations / follow-ups

- Stage inference and cache-load percent are best-effort and depend on what `@mlc-ai/web-runtime` includes in `report.text`.
- If/when `@mlc-ai/web-runtime` exposes first-class cache progress, the normalizer can become a thin pass-through while keeping `stage` semantics.

