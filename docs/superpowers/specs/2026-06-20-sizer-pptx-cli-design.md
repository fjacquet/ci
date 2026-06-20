# Sizer PPTX CLI — Design Spec

- **Date:** 2026-06-20
- **Status:** Approved (design)
- **Owner:** Fred (fjacquet)

## Goal

A "shortcut" for the sizer frontends: a command-line tool that takes the **same source
file that starts the app** (an RVTools or LiveOptics `.xlsx` export) and generates the
**light-mode PowerPoint deck** headlessly — no browser, no UI clicks. One native CLI per
app, reusing that app's existing engines.

## Scope

- **In (4 apps):** `vsizer`, `ppdm-report`, `presizion`, `vatlas`.
- **Out (this pass):** `360gantt` — its pptx export reads the rendered DOM (`ganttRef`) to
  capture the Gantt visual, so it can't go fully native; deferred (would use a
  Playwright-automation approach later).
- **Out:** other frontends without an xlsx→pptx pipeline.

## Approach (decided)

**C — Hybrid, reduced to native-only.** With the one browser-coupled app (360gantt)
dropped, all four remaining apps reuse their existing DOM-free engines in a thin Node CLI.
No Playwright, no headless browser.

Rejected alternatives:
- **A (pure native, all 5):** 360gantt's DOM-rendered Gantt would need rebuilding as pptx
  shapes — real work, out of scope.
- **B (Playwright everywhere):** uniform but heavy (spins a browser, serves the app, fragile
  to UI changes); unnecessary once the visual-only app is dropped.

## Architecture

Four **independent native Node CLIs**, one per app, each living **inside its own repo**
(`src/cli/`) and importing that app's existing engines. **No shared cross-repo package**
(YAGNI — the common scaffold is ~40 lines; duplicating beats a published dependency).

### Canonical pipeline (identical across all four)

```
read source file from disk
  → app parser            (parseXlsx / sheet parse — already DOM-free)
  → app aggregation        (sizing engines — already DOM-free)
  → assembleBuilderInput   (NEW hoisted engine fn — see below)
  → buildPptx(input, theme='light')   (existing builder, returns ArrayBuffer)
  → write ArrayBuffer to <out>.pptx
```

### The one refactor per app (assembly hoist)

Today each app assembles the pptx builder's input **inside a React hook/component**
(`src/hooks/useExport.ts` for vsizer/ppdm-report; `src/components/step3/Step3ReviewExport.tsx`
for presizion). We **hoist that pure logic into an engine function** (e.g.
`src/engines/export/assembleExportModel.ts`) that both the existing hook **and** the new CLI
import. The hook becomes a thin caller; in-app behavior is unchanged. This is a small,
well-bounded improvement, not a rewrite.

## Per-app specifics

| app | input | builder entry | refactor | theme |
|---|---|---|---|---|
| **ppdm-report** | xlsx | `buildPptx(model, theme)` — takes `'light'`/`'dark'` | hoist `ExportModel` assembly out of `useExport.ts` | native `LIGHT`/`DARK` palettes |
| **vsizer** | xlsx (RVTools/LiveOptics) | `buildPptx(input): ArrayBuffer` | hoist `BuildPptxInput` assembly out of `useExport.ts` | single light theme (`theme.ts`) |
| **presizion** | xlsx | `exportPptx()` (calls `pptx.writeFile`) | split into `assemble` + `buildPptx(): ArrayBuffer`; CLI writes the file | single light theme |
| **vatlas** | xlsx | worker → `buildPptx(view, trends, strings, locale, …)` | call `buildExportView` + `buildChartBundle` + `buildPptx` directly, bypassing `export.worker.ts` | has theme arg |

### vatlas feasibility risk — chart rasterization (SPIKE)

`src/engines/export/chartBundle.ts` rasterizes charts to images; in headless Node there is no
`<canvas>`/DOM. Before committing vatlas's CLI, run a short **spike** into how
`buildChartBundle` rasterizes:
- **Offscreen-canvas / Node-capable renderer** → add `@napi-rs/canvas` (or `node-canvas`) and
  the native CLI works.
- **Captured from already-rendered DOM** → vatlas can't go fully native; defer it (Playwright
  fallback, like 360gantt) or ship its decks without those raster charts.

**Sequencing:** build the three clean apps first; vatlas's CLI is gated on the spike outcome
and must not block them.

## CLI interface (identical across the four)

```
<app>-pptx <source-file> [--out <path>] [--theme light|dark] [--quiet]
```

- `<source-file>` — required; the RVTools/LiveOptics `.xlsx`.
- `--out` — defaults to `<source-basename>.pptx` beside the input.
- `--theme` — defaults to **`light`**; `dark` accepted only where the app has a dark palette
  (ppdm-report, vatlas), otherwise rejected with a clear message.
- Exit `0` on success (prints output path unless `--quiet`); non-zero with a readable error on
  missing/unreadable/unparseable input.

## Distribution & run

Each repo adds a `bin` entry in `package.json` plus an npm script (`npm run pptx -- <file>`).
The CLI is TypeScript executed via **`tsx`** (added as a devDependency) — no separate build
step; it imports the app's existing TS modules directly. Local/dev tool; **not published to npm**.

## Testing

Reuse each app's existing engine tests + input fixtures (e.g. vsizer's
`src/test/fixtures/buildXlsx.ts`). Each CLI adds:
- a **unit test** on the hoisted `assemble…` function (fixture-in → model-out, pure).
- one **integration test**: fixture file in → assert a valid `.pptx` out (zip `PK` magic bytes,
  non-empty, expected slide count via existing builder assertions).

No new CI surface — runs under each app's existing `make test` / vitest.

## Deliverables

- `vsizer`, `ppdm-report`, `presizion`: hoisted `assemble…` engine fn + `src/cli/` + `bin` +
  npm script + tests. One PR per repo, scoped to engines/cli/tests — the hoist preserves
  existing in-app behavior (no UI/output change).
- `vatlas`: spike doc on chart rasterization → then CLI or deferral note.

## Out of scope

- 360gantt CLI (DOM-coupled visual).
- Publishing CLIs to npm; a shared cross-repo CLI package.
- New deck designs or theme work beyond selecting the existing light palette.
