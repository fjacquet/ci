# vsizer PPTX CLI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a headless Node CLI to `vsizer` that turns an RVTools/LiveOptics source file into the light-mode `.pptx` deck the app produces, reusing the existing engines.

**Architecture:** Three small refactors hoist pure logic out of React (ingest orchestration, pptx strings, builder-input assembly) into engine functions that both the existing hooks and the new CLI import. A thin `src/cli/pptx.ts` then runs `read file → ingestDataset → buildPptxStrings → assembleBuildPptxInput → buildPptx → write file`. No UI/output behavior changes.

**Tech Stack:** TypeScript, `tsx` (run TS directly, no build step), existing `xlsx` + `pptxgenjs` engines, `i18next` (standalone instance for strings), `vitest`.

## Global Constraints

- **No UI/output behavior change.** Each hoist must leave the React hook producing the exact same result; the in-app deck is byte-identical to today.
- **Reuse engines, add no parsing/pptx logic.** The CLI orchestrates existing functions only.
- **Light theme.** vsizer has a single light theme in `src/engines/export/pptx/theme.ts`; there is no `--theme` flag for vsizer.
- **Default deck language `fr`** (the app's `fallbackLng` in `src/i18n/index.ts`); `--lang <code>` overrides.
- **Run via `tsx`**, no separate build; CLI is a dev/local tool, not published.
- **TDD + commit per task.** Each task ends green and committed.
- Builder entry is `buildPptx(input: BuildPptxInput): Promise<ArrayBuffer>` — the CLI writes that ArrayBuffer to disk via `Buffer.from(arrayBuffer)`.

---

### Task 1: Hoist dataset ingest into a pure engine function

Pull the per-file parse→aggregate loop out of `useDatasetUpload.ts` (which mixes it with `File.arrayBuffer()`, toasts, and the store) into a pure `ingestDataset` that the hook and the CLI both call.

**Files:**
- Create: `src/engines/ingest.ts`
- Create: `src/engines/ingest.test.ts`
- Modify: `src/hooks/useDatasetUpload.ts` (call the new function)

**Interfaces:**
- Consumes (existing): `extractWorkbookBytes(buffer, fileName)`, `parseDataset(buffer): { source, vinfo, vhost, errors }` (`src/engines/parser/normalizeColumns.ts`), `resolveClusterCollisions(perFile: FileScopedRows[]): { vinfo, vhost }`, `aggregateClusters({ vinfo, vhost, stretchedClusters }): ClusterAggregate[]`, `aggregateGlobals(clusters): GlobalSummary`.
- Produces:
  ```ts
  export interface IngestFile { name: string; size?: number; bytes: ArrayBuffer | Uint8Array }
  export interface IngestResult {
    sources: SourceFile[]
    source: SourceFormat
    vinfo: VInfoRow[]
    vhost: VHostRow[]
    aggregates: Record<string, ClusterAggregate>
    globals: GlobalSummary
    parseErrors: Array<{ file: string; sheet: 'vinfo' | 'vhost'; index: number; message: string }>
  }
  export function ingestDataset(files: IngestFile[], stretchedClusters?: ReadonlySet<string>): IngestResult
  ```
  Throws `IngestError` (new, extends `Error`) when every file fails or zero clusters result, with a message naming the cause.

- [ ] **Step 1: Write the failing test**

```ts
// src/engines/ingest.test.ts
import { describe, expect, it } from 'vitest'
import { readFileSync } from 'node:fs'
import { ingestDataset, IngestError } from './ingest'

const rvtools = (): ArrayBuffer => {
  // Reuse the existing fixture builder so the test data matches the parser's expectations.
  const { buildRvToolsXlsx } = require('../test/fixtures/buildXlsx') as typeof import('../test/fixtures/buildXlsx')
  return buildRvToolsXlsx()
}

describe('ingestDataset', () => {
  it('produces aggregates + globals from one RVTools workbook', () => {
    const res = ingestDataset([{ name: 'estate.xlsx', bytes: rvtools() }])
    expect(res.source).toBe('rvtools')
    expect(Object.keys(res.aggregates).length).toBeGreaterThan(0)
    expect(res.globals).not.toBeNull()
    expect(res.sources[0]?.name).toBe('estate.xlsx')
  })

  it('throws IngestError when no file parses to a known source', () => {
    expect(() => ingestDataset([{ name: 'junk.xlsx', bytes: new Uint8Array([1, 2, 3]) }])).toThrow(IngestError)
  })
})
```

(If `buildRvToolsXlsx` is not the exact export name, open `src/test/fixtures/buildXlsx.ts` and use the actual fixture export; the file is the project's canonical xlsx fixture builder.)

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/engines/ingest.test.ts`
Expected: FAIL — `Cannot find module './ingest'`.

- [ ] **Step 3: Write the engine function**

```ts
// src/engines/ingest.ts
import { aggregateClusters } from './aggregation/aggregateClusters'
import { aggregateGlobals } from './aggregation/globals'
import type { ClusterAggregate } from './aggregation/aggregateClusters'
import type { GlobalSummary } from './aggregation/globals'
import { extractWorkbookBytes } from './parser/extractWorkbook'
import { parseDataset } from './parser/normalizeColumns'
import { resolveClusterCollisions, type FileScopedRows } from './parser/resolveClusterCollisions'
import type { SourceFormat } from './parser/detectSource'
import type { VInfoRow, VHostRow } from './parser/schemas'
import type { SourceFile } from '../store/datasetStore'

export class IngestError extends Error {}

export interface IngestFile { name: string; size?: number; bytes: ArrayBuffer | Uint8Array }

export interface IngestResult {
  sources: SourceFile[]
  source: SourceFormat
  vinfo: VInfoRow[]
  vhost: VHostRow[]
  aggregates: Record<string, ClusterAggregate>
  globals: GlobalSummary
  parseErrors: Array<{ file: string; sheet: 'vinfo' | 'vhost'; index: number; message: string }>
}

export function ingestDataset(
  files: IngestFile[],
  stretchedClusters: ReadonlySet<string> = new Set(),
): IngestResult {
  const perFile: FileScopedRows[] = []
  const sources: SourceFile[] = []
  const parseErrors: IngestResult['parseErrors'] = []

  for (const file of files) {
    const workbookBytes = extractWorkbookBytes(file.bytes, file.name)
    const parsed = parseDataset(workbookBytes)
    if (parsed.source === 'unknown') continue
    perFile.push({ filename: file.name, vinfo: parsed.vinfo, vhost: parsed.vhost })
    sources.push({
      name: file.name,
      size: file.size ?? 0,
      source: parsed.source,
      vinfoRows: parsed.vinfo.length,
      vhostRows: parsed.vhost.length,
    })
    for (const err of parsed.errors) {
      parseErrors.push({ file: file.name, sheet: err.sheet, index: err.index, message: err.message })
    }
  }

  if (perFile.length === 0) throw new IngestError('No file parsed to a known RVTools/LiveOptics source')

  const { vinfo, vhost } = resolveClusterCollisions(perFile)
  const clusters = aggregateClusters({ vinfo, vhost, stretchedClusters })
  if (clusters.length === 0) throw new IngestError('No clusters found in the dataset')

  const aggregates: Record<string, ClusterAggregate> = {}
  for (const cluster of clusters) aggregates[cluster.cluster] = cluster

  return {
    sources,
    source: sources[0]!.source,
    vinfo,
    vhost,
    aggregates,
    globals: aggregateGlobals(clusters),
    parseErrors,
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/engines/ingest.test.ts`
Expected: PASS (both cases).

- [ ] **Step 5: Refactor `useDatasetUpload.ts` to call `ingestDataset`**

In `src/hooks/useDatasetUpload.ts`, replace the inline per-file loop + `resolveClusterCollisions`/`aggregateClusters`/`aggregateGlobals` block with: build `IngestFile[]` via `await file.arrayBuffer()`, call `ingestDataset(ingestFiles, useDatasetStore.getState().stretchedClusters)` inside a `try/catch`, map `IngestError` to the existing toast (`validation:rows.noClusters` / `validation:source.unknown` as appropriate), and feed the result to `setMergedDataset`. Keep the per-file `unknown`-source and zip-extract toasts by inspecting `result.sources` vs the input files. Do not change any user-visible behavior.

- [ ] **Step 6: Run the upload hook tests + typecheck**

Run: `npx vitest run src/hooks src/engines && npm run typecheck`
Expected: PASS, no type errors.

- [ ] **Step 7: Commit**

```bash
git add src/engines/ingest.ts src/engines/ingest.test.ts src/hooks/useDatasetUpload.ts
git commit -m "refactor(engines): hoist dataset ingest out of useDatasetUpload into pure ingestDataset"
```

---

### Task 2: Extract pptx strings into a pure function + a standalone i18n helper

`usePptxStrings` is a React hook (`useTranslation`). Split its body into a pure `buildPptxStrings(t, …)` so the CLI can supply a `t` from a standalone i18next instance.

**Files:**
- Create: `src/engines/export/pptx/strings.ts`
- Create: `src/engines/export/pptx/strings.test.ts`
- Create: `src/cli/i18n.ts`
- Modify: `src/hooks/usePptxStrings.ts` (delegate to `buildPptxStrings`)

**Interfaces:**
- Consumes: `PptxStrings` (from `builder.ts`), `SourceFormat`, `resources` + namespace list from `src/i18n/index.ts`, `i18next`.
- Produces:
  ```ts
  // strings.ts
  import type { TFunction } from 'i18next'
  export function buildPptxStrings(
    t: TFunction, sourceFile: string, dateIso: string, sourceFormat: SourceFormat,
  ): PptxStrings
  // cli/i18n.ts
  export function createPptxT(lng: string): TFunction   // standalone instance, namespace 'pptx'
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/engines/export/pptx/strings.test.ts
import { describe, expect, it } from 'vitest'
import { buildPptxStrings } from './strings'
import { createPptxT } from '../../../cli/i18n'

describe('buildPptxStrings', () => {
  it('fills the deck strings from a standalone i18n instance', () => {
    const t = createPptxT('fr')
    const s = buildPptxStrings(t, 'estate.xlsx', '2026-06-20', 'rvtools')
    expect(typeof s.deckTitle).toBe('string')
    expect(s.deckTitle.length).toBeGreaterThan(0)
    expect(typeof s.title.title).toBe('string')
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/engines/export/pptx/strings.test.ts`
Expected: FAIL — `Cannot find module './strings'`.

- [ ] **Step 3: Write `cli/i18n.ts` (standalone instance)**

```ts
// src/cli/i18n.ts
import i18next, { type TFunction } from 'i18next'
import { resources } from '../i18n'

/** A React-free i18next instance bound to the 'pptx' namespace, for the CLI. */
export function createPptxT(lng: string): TFunction {
  const instance = i18next.createInstance()
  instance.init({ resources, lng, fallbackLng: 'fr', ns: ['pptx'], defaultNS: 'pptx', initImmediate: false })
  return instance.getFixedT(lng, 'pptx')
}
```

(If `resources` is not exported from `src/i18n/index.ts`, add `export` to the existing `const resources` there — it is already a named const.)

- [ ] **Step 4: Write `strings.ts` by moving the hook body**

Move the entire return-object construction from `usePptxStrings` into `buildPptxStrings(t, sourceFile, dateIso, sourceFormat)`, replacing the hook's `const { t } = useTranslation('pptx')` with the injected `t` parameter. The function-typed subtitle slots stay as wrappers around `t(..., vars)`.

- [ ] **Step 5: Delegate the hook to the pure function**

```ts
// src/hooks/usePptxStrings.ts  (new body)
import { useTranslation } from 'react-i18next'
import type { PptxStrings } from '../engines/export/pptx/builder'
import type { SourceFormat } from '../engines/parser/detectSource'
import { buildPptxStrings } from '../engines/export/pptx/strings'

export function usePptxStrings(sourceFile: string, dateIso: string, sourceFormat: SourceFormat): PptxStrings {
  const { t } = useTranslation('pptx')
  return buildPptxStrings(t, sourceFile, dateIso, sourceFormat)
}
```

- [ ] **Step 6: Run tests + typecheck**

Run: `npx vitest run src/engines/export/pptx/strings.test.ts src/hooks && npm run typecheck`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/engines/export/pptx/strings.ts src/engines/export/pptx/strings.test.ts src/cli/i18n.ts src/hooks/usePptxStrings.ts src/i18n/index.ts
git commit -m "refactor(pptx): extract buildPptxStrings + standalone CLI i18n instance"
```

---

### Task 3: Hoist builder-input assembly out of useExport

Extract the pure "build `BuildPptxInput` from dataset + strings" step from `useExport.ts` into an engine function.

**Files:**
- Create: `src/engines/export/pptx/assemble.ts`
- Create: `src/engines/export/pptx/assemble.test.ts`
- Modify: `src/hooks/useExport.ts`

**Interfaces:**
- Consumes: `IngestResult` (Task 1), `PptxStrings`, `topReadinessVmsByCluster(vinfo)` (`src/engines/aggregation/vinfoMerge.ts`), `BuildPptxInput` (`builder.ts`).
- Produces:
  ```ts
  export function assembleBuildPptxInput(
    dataset: Pick<IngestResult, 'globals' | 'aggregates' | 'vhost' | 'vinfo'>,
    strings: PptxStrings,
    selectedClusters?: ReadonlySet<string>,
  ): BuildPptxInput
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/engines/export/pptx/assemble.test.ts
import { describe, expect, it } from 'vitest'
import { ingestDataset } from '../../ingest'
import { buildPptxStrings } from './strings'
import { createPptxT } from '../../../cli/i18n'
import { assembleBuildPptxInput } from './assemble'
import { buildRvToolsXlsx } from '../../../test/fixtures/buildXlsx'

describe('assembleBuildPptxInput', () => {
  it('selects all clusters sorted and wires strings + readiness', () => {
    const ds = ingestDataset([{ name: 'estate.xlsx', bytes: buildRvToolsXlsx() }])
    const strings = buildPptxStrings(createPptxT('fr'), 'estate.xlsx', '2026-06-20', ds.source)
    const input = assembleBuildPptxInput(ds, strings)
    expect(input.clusters.length).toBe(Object.keys(ds.aggregates).length)
    expect(input.strings).toBe(strings)
    expect(input.globals).toBe(ds.globals)
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/engines/export/pptx/assemble.test.ts`
Expected: FAIL — `Cannot find module './assemble'`.

- [ ] **Step 3: Write the assembly function**

```ts
// src/engines/export/pptx/assemble.ts
import { topReadinessVmsByCluster } from '../../aggregation/vinfoMerge'
import type { IngestResult } from '../../ingest'
import type { BuildPptxInput, PptxStrings } from './builder'

export function assembleBuildPptxInput(
  dataset: Pick<IngestResult, 'globals' | 'aggregates' | 'vhost' | 'vinfo'>,
  strings: PptxStrings,
  selectedClusters: ReadonlySet<string> = new Set(),
): BuildPptxInput {
  const all = Object.values(dataset.aggregates).sort((a, b) => a.cluster.localeCompare(b.cluster))
  const clusters = selectedClusters.size === 0 ? all : all.filter((c) => selectedClusters.has(c.cluster))
  return {
    globals: dataset.globals,
    clusters,
    vhost: dataset.vhost,
    topReadinessByCluster: topReadinessVmsByCluster(dataset.vinfo),
    strings,
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/engines/export/pptx/assemble.test.ts`
Expected: PASS.

- [ ] **Step 5: Refactor `useExport.ts` to call `assembleBuildPptxInput`**

In the `exportPptx` callback, replace the inline `all`/`filtered`/`buildPptx({...})` construction with:
```ts
const input = assembleBuildPptxInput({ globals, aggregates, vhost, vinfo }, strings, selectedClusters)
const data = await buildPptx(input)
```
Add `vinfo` to the store selectors already present in the hook (it currently reads `vinfo` for `topReadinessByCluster` — keep one source). Remove the now-dead `topReadinessByCluster` memo if it is only used here. Behavior unchanged.

- [ ] **Step 6: Run tests + typecheck**

Run: `npx vitest run src/engines src/hooks && npm run typecheck`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/engines/export/pptx/assemble.ts src/engines/export/pptx/assemble.test.ts src/hooks/useExport.ts
git commit -m "refactor(pptx): hoist BuildPptxInput assembly out of useExport"
```

---

### Task 4: The CLI entry point + bin + integration test

Wire the hoisted functions into a headless command.

**Files:**
- Create: `src/cli/pptx.ts`
- Create: `src/cli/pptx.test.ts`
- Modify: `package.json` (add `bin`, `scripts.pptx`, `tsx` devDependency)

**Interfaces:**
- Consumes: `ingestDataset` (Task 1), `createPptxT` + `buildPptxStrings` (Task 2), `assembleBuildPptxInput` (Task 3), `buildPptx` (existing).

- [ ] **Step 1: Add `tsx` and the script/bin to `package.json`**

Add to `devDependencies`: `"tsx": "^4.19.2"`. Add to `scripts`: `"pptx": "tsx src/cli/pptx.ts"`. Add top-level: `"bin": { "vsizer-pptx": "src/cli/pptx.ts" }`.

Run: `npm install`
Expected: `tsx` installed, lockfile updated.

- [ ] **Step 2: Write the failing integration test**

```ts
// src/cli/pptx.test.ts
import { describe, expect, it } from 'vitest'
import { mkdtempSync, writeFileSync, readFileSync } from 'node:fs'
import { tmpdir } from 'node:os'
import { join } from 'node:path'
import { runCli } from './pptx'
import { buildRvToolsXlsx } from '../test/fixtures/buildXlsx'

describe('runCli', () => {
  it('writes a valid .pptx from an RVTools file', async () => {
    const dir = mkdtempSync(join(tmpdir(), 'vsizer-cli-'))
    const input = join(dir, 'estate.xlsx')
    writeFileSync(input, Buffer.from(buildRvToolsXlsx()))
    const out = join(dir, 'out.pptx')
    const code = await runCli(['--out', out, '--quiet', input])
    expect(code).toBe(0)
    const bytes = readFileSync(out)
    expect(bytes.length).toBeGreaterThan(1000)
    expect(bytes.subarray(0, 2).toString('latin1')).toBe('PK') // pptx is a zip
  })

  it('returns non-zero on a missing file', async () => {
    expect(await runCli(['/no/such/file.xlsx', '--quiet'])).not.toBe(0)
  })
})
```

- [ ] **Step 3: Run test to verify it fails**

Run: `npx vitest run src/cli/pptx.test.ts`
Expected: FAIL — `Cannot find module './pptx'`.

- [ ] **Step 4: Write the CLI**

```ts
// src/cli/pptx.ts
import { readFile, writeFile } from 'node:fs/promises'
import { basename, dirname, join } from 'node:path'
import { ingestDataset, IngestError } from '../engines/ingest'
import { buildPptx } from '../engines/export/pptx/builder'
import { assembleBuildPptxInput } from '../engines/export/pptx/assemble'
import { buildPptxStrings } from '../engines/export/pptx/strings'
import { createPptxT } from './i18n'

interface Args { input?: string; out?: string; lang: string; quiet: boolean }

function parseArgs(argv: string[]): Args {
  const args: Args = { lang: 'fr', quiet: false }
  for (let i = 0; i < argv.length; i++) {
    const a = argv[i]
    if (a === '--out') args.out = argv[++i]
    else if (a === '--lang') args.lang = argv[++i] ?? 'fr'
    else if (a === '--quiet') args.quiet = true
    else if (!a.startsWith('-')) args.input = a
  }
  return args
}

const todayIso = (): string => new Date().toISOString().slice(0, 10)

export async function runCli(argv: string[]): Promise<number> {
  const args = parseArgs(argv)
  if (!args.input) {
    process.stderr.write('usage: vsizer-pptx <source.xlsx> [--out file] [--lang code] [--quiet]\n')
    return 2
  }
  try {
    const bytes = await readFile(args.input)
    const ds = ingestDataset([{ name: basename(args.input), size: bytes.length, bytes }])
    const strings = buildPptxStrings(createPptxT(args.lang), basename(args.input), todayIso(), ds.source)
    const input = assembleBuildPptxInput(ds, strings)
    const deck = await buildPptx(input)
    const out = args.out ?? join(dirname(args.input), `${basename(args.input).replace(/\.[^.]+$/, '')}_vsizer.pptx`)
    await writeFile(out, Buffer.from(deck))
    if (!args.quiet) process.stdout.write(`${out}\n`)
    return 0
  } catch (err) {
    const msg = err instanceof Error ? err.message : String(err)
    process.stderr.write(`${err instanceof IngestError ? 'ingest error' : 'error'}: ${msg}\n`)
    return 1
  }
}

// Node passes [node, script, ...args]; strip the first two.
if (process.argv[1] && process.argv[1].endsWith('pptx.ts')) {
  runCli(process.argv.slice(2)).then((code) => process.exit(code))
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run src/cli/pptx.test.ts`
Expected: PASS (both cases).

- [ ] **Step 6: Smoke-test the real command**

Run: `npm run pptx -- $(node -e "const {buildRvToolsXlsx}=require('./src/test/fixtures/buildXlsx'); const fs=require('fs'); const p='/tmp/estate.xlsx'; fs.writeFileSync(p, Buffer.from(buildRvToolsXlsx())); process.stdout.write(p)")`
Expected: prints a path ending `_vsizer.pptx`; the file opens in PowerPoint/Keynote and shows the title + per-cluster slides in the light theme.

(If the fixture builder isn't CommonJS-requireable, instead point the command at any real RVTools `.xlsx` you have on disk.)

- [ ] **Step 7: Full gate + commit**

```bash
npm run typecheck && npx vitest run && npm run lint
git add src/cli/pptx.ts src/cli/pptx.test.ts package.json package-lock.json
git commit -m "feat(cli): vsizer-pptx — RVTools/LiveOptics source file to light-mode pptx"
```

---

## Self-Review

- **Spec coverage:** canonical pipeline (Task 4 wiring) ✓; assembly hoist (Task 3) ✓; ingest reuse (Task 1) ✓; light theme default — vsizer single theme, no `--theme` (Global Constraints) ✓; CLI interface `<file> [--out] [--quiet]` + `--lang` for the fr-default i18n (Task 4) ✓; testing = unit on hoisted fns + integration asserting PK/zip (Tasks 1–4) ✓; `tsx`/`bin`/no-publish (Task 4) ✓. vatlas/ppdm-report/presizion are separate plans (out of scope here, per spec sequencing).
- **Placeholder scan:** all code steps contain real code; the two "if the fixture export name differs" notes point at a concrete file to read, not a TODO. ✓
- **Type consistency:** `ingestDataset → IngestResult` consumed by `assembleBuildPptxInput`; `createPptxT → TFunction` consumed by `buildPptxStrings`; `assembleBuildPptxInput → BuildPptxInput` consumed by `buildPptx`. Names match across tasks. ✓
