# ppdm-report PPTX CLI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a headless `ppdm-report-pptx` CLI that turns a PPDM/LiveOptics source `.xlsx` into the report deck, reusing the existing engines.

**Architecture:** ppdm-report already factors the deck assembly (`buildExportModel`) and theming (`buildPptx(model, theme)`) as pure engine functions, so this is smaller than vsizer's: hoist a synchronous ingest (parse → `buildEstateDocument`, bypassing the parse Web Worker), add a React-free i18n `t`, then a thin CLI. No assembly hoist is needed.

**Tech Stack:** TypeScript, `tsx`, existing `xlsx` + `pptxgenjs` engines, `i18next` (standalone instance), `vitest`, biome.

## Global Constraints

- **The gate is `npm run build` (NOT just `npm run typecheck`).** The production build (`tsc -b && vite build`) type-checks via project references and is stricter than `tsc --noEmit`; it is what CI runs. Every task must end with `npm run build` green. (This is the lesson from the vsizer CLI, whose CI failed because the CLI used Node globals the app tsconfig didn't type.)
- **Keep the CLI out of the app bundle.** `src/cli/**` must be excluded from `tsconfig.app.json` and type-checked instead via `tsconfig.node.json` (which already carries Node types for `vite.config.ts`); add `@types/node` to devDependencies if missing.
- **No parameter-property constructors** anywhere reachable from the app build — `erasableSyntaxOnly` rejects `constructor(public readonly x: …)`. Use explicit field declarations.
- **No in-app behavior change** — refactors must leave the hooks/worker producing the same result.
- Defaults: theme **`light`**, flavor **`assessment`** (store default), language **`en`** (i18n `fallbackLng`). ppdm-report HAS a dark palette, so `--theme light|dark` is valid here.
- Run via `tsx`; CLI is a dev/local tool, not published; it makes no network calls (uphold the in-browser/no-exfiltration product invariant — the CLI runs locally only).
- TDD + commit per task. Confirm `git branch --show-current` is the feature branch before each commit; never switch branches.

---

### Task 1: Hoist a synchronous ingest (`ingestReport`)

The app parses workbooks in a Web Worker (`src/engines/parser/parser.worker.ts` via `parseInWorker`) and derives the estate with `buildEstateDocument`. Extract the worker's sync parse core so the CLI can parse on the main thread, and provide a one-call ingest.

**Files:**
- Create: `src/engines/ingestReport.ts`
- Create: `src/engines/ingestReport.test.ts`
- Modify: `src/engines/parser/parser.worker.ts` (delegate to the shared core) — only if it currently inlines the parse; otherwise leave it.

**Interfaces:**
- Consumes (existing): `readWorkbook(buf: ArrayBuffer): WorkBook` / `parseXlsx(buf): SheetData[]` (`src/engines/parser/readWorkbook.ts`); whatever turns a parsed workbook + label into a `ServerWorkbook` (read `parseInWorker.ts` / `parser.worker.ts` to find it — it is the sync logic the worker runs); `buildEstateDocument(servers: ServerWorkbook[]): EstateDocument` (`src/engines/products/estateDocument.ts`).
- Produces:
  ```ts
  export interface ReportFile { name: string; bytes: ArrayBuffer | Uint8Array }
  export function parseServerWorkbook(name: string, bytes: ArrayBuffer | Uint8Array): ServerWorkbook
  export function ingestReport(files: ReportFile[]): EstateDocument   // = buildEstateDocument(files.map(parseServerWorkbook))
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/engines/ingestReport.test.ts
import { describe, expect, it } from 'vitest'
import { ingestReport } from './ingestReport'
import { summaryWorkbookBuffer } from '../test-helpers/workbooks'

describe('ingestReport', () => {
  it('parses a PPDM summary workbook into an EstateDocument with one product', () => {
    const doc = ingestReport([{ name: 'acme_ppdm.xlsx', bytes: summaryWorkbookBuffer() }])
    expect(doc.products.length).toBeGreaterThan(0)
    expect(doc.products[0]?.estate.combined).toBeTruthy()
  })
})
```

- [ ] **Step 2: Run test to verify it fails** — `npx vitest run src/engines/ingestReport.test.ts` → `Cannot find module './ingestReport'`.

- [ ] **Step 3: Implement `ingestReport.ts`.** Read `parser.worker.ts`/`parseInWorker.ts` to learn the exact sync transform from `(name, ArrayBuffer)` to `ServerWorkbook` (label from filename, `readWorkbook` for the workbook, plus any meta capture the worker does). Put that in `parseServerWorkbook`, then `ingestReport = (files) => buildEstateDocument(files.map(f => parseServerWorkbook(f.name, f.bytes)))`. No React/store/DOM imports. If `parser.worker.ts` had that logic inline, refactor it to import `parseServerWorkbook` so worker and CLI share one implementation.

- [ ] **Step 4: Run test to verify it passes** — `npx vitest run src/engines/ingestReport.test.ts` → PASS.

- [ ] **Step 5: Build gate** — `npm run build && npx vitest run` → both green (the worker still compiles; estate output unchanged).

- [ ] **Step 6: Commit**

```bash
git add src/engines/ingestReport.ts src/engines/ingestReport.test.ts src/engines/parser/parser.worker.ts
git commit -m "refactor(engines): synchronous ingestReport reusing the worker parse core"
```

---

### Task 2: React-free i18n translator for the CLI

`buildExportModel` already takes a `t: (key, opts?) => string` (resolving `ns:key` like `dashboard:perServer.title`, `common:sizeUnknown`) and a `locale`. Provide a standalone instance so the CLI can supply `t` without React.

**Files:**
- Create: `src/cli/i18n.ts`
- Create: `src/cli/i18n.test.ts`

**Interfaces:**
- Consumes: `resources`, `NAMESPACES`, from `src/i18n/index.ts` (already exported; `fallbackLng: 'en'`, `NAMESPACES = ['common','dashboard']`).
- Produces: `export function createReportT(lng: string): (key: string, opts?: Record<string, unknown>) => string`

- [ ] **Step 1: Write the failing test**

```ts
// src/cli/i18n.test.ts
import { describe, expect, it } from 'vitest'
import { createReportT } from './i18n'

describe('createReportT', () => {
  it('resolves namespaced keys without React', () => {
    const t = createReportT('en')
    expect(typeof t('common:sizeUnknown')).toBe('string')
    expect(t('common:sizeUnknown').length).toBeGreaterThan(0)
  })
})
```

- [ ] **Step 2: Run test to verify it fails** — `npx vitest run src/cli/i18n.test.ts`.

- [ ] **Step 3: Implement**

```ts
// src/cli/i18n.ts
import i18next from 'i18next'
import { NAMESPACES, resources } from '../i18n'

/** React-free i18next translator resolving `ns:key`, for the CLI. */
export function createReportT(lng: string): (key: string, opts?: Record<string, unknown>) => string {
  const instance = i18next.createInstance()
  // i18next v26: no initImmediate option; init is synchronous with inline resources.
  void instance.init({ resources, lng, fallbackLng: 'en', ns: [...NAMESPACES] })
  return (key, opts) => instance.t(key, opts) as string
}
```

(Confirm the installed i18next version's init signature while implementing — the vsizer CLI hit an `initImmediate` removal in i18next v26. Do not pass options the type rejects.)

- [ ] **Step 4: Run test to verify it passes** — `npx vitest run src/cli/i18n.test.ts` → PASS.

- [ ] **Step 5: Build gate** — `npm run build` → green (note: `src/cli` must be wired into `tsconfig.node.json`; if Task 3 hasn't done that yet, this step may surface the Node-types gap early — if so, do the tsconfig wiring from Task 3 Step 1 now).

- [ ] **Step 6: Commit**

```bash
git add src/cli/i18n.ts src/cli/i18n.test.ts
git commit -m "feat(cli): standalone i18next translator for headless export"
```

---

### Task 3: The CLI entry point, bin, and build wiring

**Files:**
- Modify: `tsconfig.app.json` (exclude `src/cli`), `tsconfig.node.json` (include `src/cli`), `package.json` (bin, script, `tsx` + `@types/node` devDeps)
- Create: `src/cli/pptx.ts`, `src/cli/pptx.test.ts`

**Interfaces consumed:** `ingestReport` (Task 1), `createReportT` (Task 2), `buildExportModel(view, flavor, theme, t, locale, perServer): ExportModel` (`src/engines/export/buildExportModel.ts`), `buildPptx(model, theme): Promise<ArrayBuffer>` (`src/engines/export/pptx/builder.ts`). Estate selection mirrors the app: `doc.products[0].estate` (Phase-1 sole PPDM section), with `estate.combined` and `estate.perServer`.

- [ ] **Step 1: Build wiring first (so the CLI compiles in CI).** In `tsconfig.app.json` add `"src/cli"` to `exclude`. In `tsconfig.node.json` add `"src/cli/**/*"` to `include`. Add `tsx` and `@types/node` to `devDependencies`; add `"pptx": "tsx src/cli/pptx.ts"` to scripts and `"bin": { "ppdm-report-pptx": "src/cli/pptx.ts" }`. Run `npm install`. Then `npm run build` must still pass.

- [ ] **Step 2: Write the failing integration test**

```ts
// src/cli/pptx.test.ts
import { describe, expect, it } from 'vitest'
import { mkdtempSync, writeFileSync, readFileSync } from 'node:fs'
import { tmpdir } from 'node:os'
import { join } from 'node:path'
import { runCli } from './pptx'
import { summaryWorkbookBuffer } from '../test-helpers/workbooks'

describe('runCli', () => {
  it('writes a valid .pptx from a PPDM workbook', async () => {
    const dir = mkdtempSync(join(tmpdir(), 'ppdm-cli-'))
    const input = join(dir, 'acme.xlsx')
    writeFileSync(input, Buffer.from(summaryWorkbookBuffer()))
    const out = join(dir, 'out.pptx')
    expect(await runCli(['--out', out, '--quiet', input])).toBe(0)
    const bytes = readFileSync(out)
    expect(bytes.length).toBeGreaterThan(1000)
    expect(bytes.subarray(0, 2).toString('latin1')).toBe('PK')
  })
  it('returns non-zero on a missing file', async () => {
    expect(await runCli(['/no/such/file.xlsx', '--quiet'])).not.toBe(0)
  })
})
```

- [ ] **Step 3: Run test to verify it fails** — `npx vitest run src/cli/pptx.test.ts`.

- [ ] **Step 4: Implement `src/cli/pptx.ts`**

```ts
import { readFile, writeFile } from 'node:fs/promises'
import { basename, dirname, join } from 'node:path'
import { ingestReport } from '../engines/ingestReport'
import { buildExportModel } from '../engines/export/buildExportModel'
import { buildPptx } from '../engines/export/pptx/builder'
import { createReportT } from './i18n'

interface Args { input?: string; out?: string; lang: string; theme: 'light' | 'dark'; flavor: 'assessment' | 'ops'; quiet: boolean }

function parseArgs(argv: string[]): Args {
  const a: Args = { lang: 'en', theme: 'light', flavor: 'assessment', quiet: false }
  for (let i = 0; i < argv.length; i++) {
    const x = argv[i]
    if (x === '--out') a.out = argv[++i]
    else if (x === '--lang') a.lang = argv[++i] ?? 'en'
    else if (x === '--theme') a.theme = argv[++i] === 'dark' ? 'dark' : 'light'
    else if (x === '--flavor') a.flavor = argv[++i] === 'ops' ? 'ops' : 'assessment'
    else if (x === '--quiet') a.quiet = true
    else if (x && !x.startsWith('-')) a.input = x
  }
  return a
}

export async function runCli(argv: string[]): Promise<number> {
  const args = parseArgs(argv)
  if (!args.input) { process.stderr.write('usage: ppdm-report-pptx <source.xlsx> [--out f] [--lang c] [--theme light|dark] [--flavor assessment|ops] [--quiet]\n'); return 2 }
  try {
    const bytes = await readFile(args.input)
    const doc = ingestReport([{ name: basename(args.input), bytes }])
    const estate = doc.products[0]?.estate
    if (!estate) { process.stderr.write('error: no product estate produced from the input\n'); return 1 }
    const t = createReportT(args.lang)
    const model = buildExportModel(estate.combined, args.flavor, args.theme, t, args.lang, estate.perServer)
    const deck = await buildPptx(model, args.theme)
    const out = args.out ?? join(dirname(args.input), `${basename(args.input).replace(/\.[^.]+$/, '')}_ppdm-report.pptx`)
    await writeFile(out, Buffer.from(deck))
    if (!args.quiet) process.stdout.write(`${out}\n`)
    return 0
  } catch (err) {
    process.stderr.write(`error: ${err instanceof Error ? err.message : String(err)}\n`)
    return 1
  }
}

if (process.argv[1]?.endsWith('pptx.ts')) { runCli(process.argv.slice(2)).then((c) => process.exit(c)) }
```

(Confirm the `flavor` argument type matches `buildExportModel`'s `ExportFlavor`; the store's `Flavor` is `'assessment' | 'ops'`. If `ExportFlavor` differs, adapt the flag values to it.)

- [ ] **Step 5: Run test to verify it passes** — `npx vitest run src/cli/pptx.test.ts` → PASS.

- [ ] **Step 6: Full gate** — `npm run build && npx vitest run && npm run lint`. All green. Biome-clean the new files if needed (`npx biome check --write src/cli/pptx.ts src/cli/pptx.test.ts`).

- [ ] **Step 7: Commit**

```bash
git add tsconfig.app.json tsconfig.node.json package.json package-lock.json src/cli/pptx.ts src/cli/pptx.test.ts
git commit -m "feat(cli): ppdm-report-pptx — source file to report deck"
```

---

### Task 4: Document the CLI

**Files:** Modify `README.md`, `CLAUDE.md`.

- [ ] **Step 1:** Add a `npm run pptx -- <file>` row to the README Scripts table and a short **CLI (`ppdm-report-pptx`)** section: usage, the `--out/--lang/--theme/--flavor/--quiet` flags, the `light`/`assessment`/`en` defaults, the `ingestReport → buildExportModel → buildPptx` reuse (deck identical to the app), and the local-only no-network note.

- [ ] **Step 2:** Add the command to the CLAUDE.md Commands block and a **Headless CLI** section mirroring the README (command, the engine modules it reuses, defaults, and that it upholds the no-exfiltration invariant).

- [ ] **Step 3: Commit**

```bash
git add README.md CLAUDE.md
git commit -m "docs: document the ppdm-report-pptx headless CLI"
```

---

## Self-Review

- **Spec coverage:** RVTools/PPDM source → deck (Task 3) ✓; light theme default + selectable dark (ppdm has DARK palette) (Task 3 flags) ✓; reuse engines, no assembly hoist needed because `buildExportModel`/`buildPptx` already factored (Architecture) ✓; standalone i18n (Task 2) ✓; ingest reuse via shared worker core (Task 1) ✓; CLI interface + tsx/bin/no-publish (Task 3) ✓; testing = unit + integration asserting PK/zip (Tasks 1–3) ✓; **docs (Task 4)** ✓; **`npm run build` in every gate** (Global Constraints) ✓.
- **Placeholder scan:** code steps carry real code; the two "confirm the exact X while implementing" notes point at concrete files (parser.worker.ts, ExportFlavor) to read, not TODOs. ✓
- **Type consistency:** `ingestReport → EstateDocument`; `doc.products[0].estate.{combined,perServer}` fed to `buildExportModel(view, flavor, theme, t, locale, perServer)`; `createReportT → TFn`; `buildExportModel → ExportModel → buildPptx(model, theme)`. Names match across tasks. ✓
