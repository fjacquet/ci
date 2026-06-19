# CI/CD Standardization — Foundation (Phases 0–2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and validate `fjacquet/ci@v1` — a central repo of SHA-pinned, best-practice reusable GitHub Actions workflows + composite actions — proven green on one pilot repo per archetype.

**Architecture:** A single public repo (`fjacquet/ci`) holds 9 reusable workflows (`workflow_call`) and 5 composite actions. Third-party actions are written with readable `@vN` tags and pinned to commit SHAs by `pinact`. Consumer repos call them as `uses: fjacquet/ci/.github/workflows/<wf>.yml@v1`. Pilots (`pscale_exporter`, `camt-csv`, `finwiz`, `vsizer`) validate each archetype end-to-end before the `v1` tag is cut.

**Tech Stack:** GitHub Actions (reusable workflows + composite actions), `actionlint` (schema lint), `zizmor` via `uvx` (security audit), `pinact` v4 (SHA pinning), `goreleaser` v2, `uv`/`ruff` (Python), `golangci-lint` v2 (Go), `npm`/`biome`/`vite` (frontend), `mkdocs-material` (docs), Dependabot.

---

## Global Constraints

Copied verbatim from `DESIGN.md`; every task implicitly includes these.

- **Repo:** all work lands in `~/Projects/ci` (remote `fjacquet/ci`, public). Work on a branch; do not push to `main` until a task says so.
- **Pinning policy:** third-party actions SHA-pinned via `pinact run` (adds `# vX.Y.Z` comment). Callers reference `fjacquet/ci` by tag `@v1` (first-party-you-control = tag is fine).
- **Per-job best-practice defaults (every job, no exceptions):**
  - `permissions:` declared explicitly, least-privilege (start from `contents: read`).
  - `concurrency: { group: <wf>-${{ github.ref }}, cancel-in-progress: true }` for CI; Pages deploys use `group: pages, cancel-in-progress: false`.
  - `timeout-minutes:` set on every job.
  - `runs-on: ubuntu-24.04` (pinned, never `-latest`).
  - First step of every job: `step-security/harden-runner@v2` with `egress-policy: audit`.
  - `actions/checkout@v6` always with `persist-credentials: false`.
- **Canonical builder versions (write these tags; pinact pins them):**
  `actions/checkout@v6`, `actions/setup-go@v6`, `actions/setup-python@v6`,
  `actions/setup-node@v6`, `astral-sh/setup-uv@v8`, `actions/upload-artifact@v7`,
  `actions/configure-pages@v6`, `actions/upload-pages-artifact@v5`,
  `actions/deploy-pages@v5`, `goreleaser/goreleaser-action@v7`,
  `golangci/golangci-lint-action@v8`, `github/codeql-action/*@v4`,
  `google/osv-scanner-action/.github/workflows/osv-scanner-reusable.yml@v2`,
  `codecov/codecov-action@v5`, `step-security/harden-runner@v2`,
  `docker/setup-qemu-action@v3`, `docker/setup-buildx-action@v3`,
  `docker/login-action@v3`, `docker/metadata-action@v5`, `docker/build-push-action@v6`.
- **Canonical secret names:** `HOMEBREW_TAP_GITHUB_TOKEN` (Go release tap), `CODECOV_TOKEN` (optional). No other secret-name variants.
- **Standard Validation Cycle (SVC)** — referenced by every authoring task. Run from `~/Projects/ci`:
  1. `actionlint` — Expected: no output (exit 0). Lints all `.github/workflows/*.yml`.
  2. `uvx zizmor --format=plain .` — Expected: `No findings` (or only known/allowlisted). Audits workflows + `action.yml` + `dependabot.yml`.
  3. `pinact run` then `pinact run --check` — first pins tags→SHAs, second Expected: exit 0 (everything pinned).
  Composite `action.yml` files are linted by `zizmor` (actionlint does not parse them); their behavioural test is the pilot run that consumes them.
- **Tooling install (Task 2 sets these up; assume present thereafter):** `brew install actionlint pinact` ; `zizmor` runs via `uvx zizmor` (no install). `gh` CLI authenticated as `fjacquet`.

---

## Task 1: Audit & inventory (Phase 0)

**Files:**
- Create: `~/Projects/ci/docs/AUDIT.md`

**Interfaces:**
- Produces: the authoritative `AUDIT.md` table consumed by the Phase 3 fleet-rollout plan and by pilot tasks (confirms archetype + current builders per repo).

- [ ] **Step 1: Generate the raw inventory**

Run from `~/Projects`:
```bash
cd ~/Projects
for d in cee-exporter ecs_exporter idrac_exporter nbu_exporter nsr_exporter \
  pflex_exporter pmax_exporter ppdd_exporter ppdm_exporter pscale_exporter pstore_exporter \
  camt-csv go-evtx pdf2md san-conv spec-search \
  finwiz classifai anki-maker mailtag lrc-automation code-review-graph Nano-Banana-MCP store-predict ppdm-report ppdm2jira vault-rag-mcp \
  elk-sizer network-sizer os-sizer vcf-sizer vsizer icons 360gantt converty llmvram presizion raidy vatlas vgpu-advisor; do
  [ -d "$d/.git" ] || { echo "$d MISSING"; continue; }
  fork=$(gh repo view "fjacquet/$d" --json isFork -q .isFork 2>/dev/null || echo "?")
  lang=$(gh repo view "fjacquet/$d" --json primaryLanguage -q .primaryLanguage.name 2>/dev/null || echo "?")
  wf=$(ls "$d/.github/workflows" 2>/dev/null | tr '\n' ',' )
  printf '%s\tfork=%s\tlang=%s\twf=%s\n' "$d" "$fork" "$lang" "$wf"
done
```
Expected: one tab-separated line per repo with fork flag, primary language, and workflow filenames.

- [ ] **Step 2: Write `docs/AUDIT.md`**

Create a markdown table from Step 1 output with columns:
`repo | archetype | fork? | primary lang | current workflows | publish target (pages-app / pages-docs / pypi / npm / none) | SAST (semgrep/codeql/none) | SBOM? | notes`.

Classify each repo into exactly one archetype: `go-exporter`, `go-cli`, `python`, `frontend`, or `excluded-fork`. Mark the ambiguous repos explicitly in the `notes` column with the resolution:
- `idrac_exporter` — set `fork?` from Step 1; if `isFork=true`, archetype = `excluded-fork`.
- `store-predict` — inspect `~/Projects/store-predict/go.mod` (exists ⇒ `go-cli`) vs `pyproject.toml` (exists ⇒ `python`); record which.
- `ppdm-report` — inspect for `deploy.yml` deploying an app to Pages (⇒ `frontend`) vs docs (⇒ `python`); record which.
- `ppdm2jira`, `vault-rag-mcp` — `python`, note "greenfield (no CI yet)".
- `vgpu-advisor`, `brave-search-mcp-server` — note "publishes to npm; needs npm-release decision (Phase 3)".

- [ ] **Step 3: Commit**

```bash
cd ~/Projects/ci
git checkout -b task-01-audit
git add docs/AUDIT.md
git commit -m "docs: add Phase 0 CI/CD audit inventory"
```

---

## Task 2: Scaffold `fjacquet/ci` repo + self-validation CI

**Files:**
- Create: `~/Projects/ci/README.md`
- Create: `~/Projects/ci/.github/workflows/self-check.yml`
- Create: `~/Projects/ci/.github/dependabot.yml`
- Create: `~/Projects/ci/zizmor.yml`
- Create: `~/Projects/ci/.gitignore`

**Interfaces:**
- Produces: `self-check.yml` (runs actionlint + zizmor + pinact-check on this repo); `zizmor.yml` forbidden-uses allowlist; `dependabot.yml` watching `github-actions`.

- [ ] **Step 1: Install tooling**

```bash
brew install actionlint pinact
uvx zizmor --version   # smoke-test zizmor is reachable via uv
```
Expected: versions print; no errors.

- [ ] **Step 2: Write `zizmor.yml` (forbidden-uses allowlist — enforces "trusted orgs only")**

```yaml
# zizmor.yml — only allow actions from trusted orgs; everything must be SHA-pinned (enforced by pinact)
rules:
  forbidden-uses:
    config:
      allow:
        - actions/*
        - github/codeql-action/*
        - astral-sh/*
        - golangci/*
        - goreleaser/*
        - codecov/*
        - docker/*
        - google/osv-scanner-action/*
        - aquasecurity/*
        - anchore/*
        - step-security/*
        - fjacquet/*
```

- [ ] **Step 3: Write `.github/dependabot.yml`**

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      actions:
        patterns: ["*"]
```

- [ ] **Step 4: Write `.gitignore`**

```gitignore
site/
dist/
*.sarif
```

- [ ] **Step 5: Write `.github/workflows/self-check.yml`**

```yaml
name: self-check
on:
  push:
    branches: [main]
  pull_request:
permissions:
  contents: read
concurrency:
  group: self-check-${{ github.ref }}
  cancel-in-progress: true
jobs:
  lint:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - name: actionlint
        run: |
          bash <(curl -sSf https://raw.githubusercontent.com/rhysd/actionlint/main/scripts/download-actionlint.bash)
          ./actionlint -color
      - uses: astral-sh/setup-uv@v8
      - name: zizmor
        run: uvx zizmor --format=github .
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: pinact check
        run: |
          go install github.com/suzuki-shunsuke/pinact/v4/cmd/pinact@latest
          "$(go env GOPATH)/bin/pinact" run --check
```

- [ ] **Step 6: Write `README.md`**

Include: purpose, the versioning policy (callers pin `@v1`; breaking changes ⇒ `v2`), a per-workflow usage table with a copy-paste caller snippet for each of the 9 workflows, and the required `permissions:` each caller must grant. (Fill the per-workflow rows as those workflows are authored in Tasks 8–16; a stub table with the 9 names is acceptable here and completed in Task 21.)

- [ ] **Step 7: Validate (SVC) and commit**

Run the SVC. `actionlint` and `zizmor` Expected: clean. Then:
```bash
cd ~/Projects/ci
git add README.md zizmor.yml .gitignore .github/
git commit -m "chore: scaffold fjacquet/ci (self-check, dependabot, zizmor allowlist)"
```

---

## Task 3: Composite action — `harden`

**Files:**
- Create: `~/Projects/ci/actions/harden/action.yml`

**Interfaces:**
- Produces: `fjacquet/ci/actions/harden` — wraps `step-security/harden-runner`. Consumed by `go-security` and `web-py-security` (Tasks 9, 16). Note: most workflows call `harden-runner` directly as their first step (per Global Constraints); this composite exists for the security workflows that want a single named entry.

- [ ] **Step 1: Write `actions/harden/action.yml`**

```yaml
name: harden
description: Apply step-security harden-runner egress audit
inputs:
  egress-policy:
    description: audit or block
    default: audit
runs:
  using: composite
  steps:
    - uses: step-security/harden-runner@v2
      with:
        egress-policy: ${{ inputs.egress-policy }}
```

- [ ] **Step 2: Validate**

```bash
cd ~/Projects/ci
uvx zizmor --format=plain actions/harden/action.yml
pinact run actions/harden/action.yml && pinact run --check actions/harden/action.yml
```
Expected: zizmor clean; pinact pins `step-security/harden-runner@v2` to a SHA and `--check` passes.

- [ ] **Step 3: Commit**

```bash
git add actions/harden/action.yml
git commit -m "feat: add harden composite action"
```

---

## Task 4: Composite action — `setup-go-cache`

**Files:**
- Create: `~/Projects/ci/actions/setup-go-cache/action.yml`

**Interfaces:**
- Produces: `fjacquet/ci/actions/setup-go-cache` — sets up Go with module/build cache. Inputs: `go-version` (string, default `""` ⇒ use `go.mod`), `cache` (string, default `"true"`). Consumed by `go-ci`, `go-release`, `go-security`.

- [ ] **Step 1: Write `actions/setup-go-cache/action.yml`**

```yaml
name: setup-go-cache
description: Set up the Go toolchain with module and build caching
inputs:
  go-version:
    description: Explicit Go version; empty uses go.mod
    default: ""
  cache:
    description: Enable build/module cache
    default: "true"
runs:
  using: composite
  steps:
    - uses: actions/setup-go@v6
      with:
        go-version: ${{ inputs.go-version }}
        go-version-file: go.mod
        cache: ${{ inputs.cache }}
```

- [ ] **Step 2: Validate**

```bash
uvx zizmor --format=plain actions/setup-go-cache/action.yml
pinact run actions/setup-go-cache/action.yml && pinact run --check actions/setup-go-cache/action.yml
```
Expected: zizmor clean; `actions/setup-go@v6` pinned.

- [ ] **Step 3: Commit**

```bash
git add actions/setup-go-cache/action.yml
git commit -m "feat: add setup-go-cache composite action"
```

---

## Task 5: Composite action — `setup-uv`

**Files:**
- Create: `~/Projects/ci/actions/setup-uv/action.yml`

**Interfaces:**
- Produces: `fjacquet/ci/actions/setup-uv` — Python + uv with cache. Inputs: `python-version` (default `"3.13"`), `enable-cache` (default `"true"`). Consumed by `python-ci`, `python-release`.

- [ ] **Step 1: Write `actions/setup-uv/action.yml`**

```yaml
name: setup-uv
description: Set up Python and uv with caching
inputs:
  python-version:
    default: "3.13"
  enable-cache:
    default: "true"
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v6
      with:
        python-version: ${{ inputs.python-version }}
    - uses: astral-sh/setup-uv@v8
      with:
        enable-cache: ${{ inputs.enable-cache }}
```

- [ ] **Step 2: Validate**

```bash
uvx zizmor --format=plain actions/setup-uv/action.yml
pinact run actions/setup-uv/action.yml && pinact run --check actions/setup-uv/action.yml
```
Expected: zizmor clean; `setup-python@v6` and `setup-uv@v8` pinned.

- [ ] **Step 3: Commit**

```bash
git add actions/setup-uv/action.yml
git commit -m "feat: add setup-uv composite action"
```

---

## Task 6: Composite action — `sbom`

**Files:**
- Create: `~/Projects/ci/actions/sbom/action.yml`

**Interfaces:**
- Produces: `fjacquet/ci/actions/sbom` — ecosystem-aware CycloneDX SBOM. Inputs: `ecosystem` (`go|python|node`, required), `output` (default `dist/sbom.cdx.json`). Consumed by `go-security`, `web-py-security`. Assumes the relevant toolchain (Go / uv-synced env / node) is already set up by the calling job.

- [ ] **Step 1: Write `actions/sbom/action.yml`**

```yaml
name: sbom
description: Generate a CycloneDX SBOM for go, python, or node
inputs:
  ecosystem:
    description: go | python | node
    required: true
  output:
    default: dist/sbom.cdx.json
runs:
  using: composite
  steps:
    - name: SBOM (go)
      if: ${{ inputs.ecosystem == 'go' }}
      shell: bash
      run: |
        mkdir -p "$(dirname "${{ inputs.output }}")"
        go run github.com/CycloneDX/cyclonedx-gomod/cmd/cyclonedx-gomod@latest \
          mod -json -output "${{ inputs.output }}"
    - name: SBOM (python)
      if: ${{ inputs.ecosystem == 'python' }}
      shell: bash
      run: |
        mkdir -p "$(dirname "${{ inputs.output }}")"
        uv run cyclonedx-py environment --output-format JSON --output-file "${{ inputs.output }}"
    - name: SBOM (node)
      if: ${{ inputs.ecosystem == 'node' }}
      shell: bash
      run: |
        mkdir -p "$(dirname "${{ inputs.output }}")"
        npx --yes @cyclonedx/cyclonedx-npm@latest --omit dev \
          --output-format JSON --output-file "${{ inputs.output }}"
    - uses: actions/upload-artifact@v7
      with:
        name: sbom-${{ inputs.ecosystem }}
        path: ${{ inputs.output }}
```

- [ ] **Step 2: Validate**

```bash
uvx zizmor --format=plain actions/sbom/action.yml
pinact run actions/sbom/action.yml && pinact run --check actions/sbom/action.yml
```
Expected: zizmor clean; `upload-artifact@v7` pinned.

- [ ] **Step 3: Commit**

```bash
git add actions/sbom/action.yml
git commit -m "feat: add ecosystem-aware sbom composite action"
```

---

## Task 7: Composite action — `mkdocs-publish`

**Files:**
- Create: `~/Projects/ci/actions/mkdocs-publish/action.yml`

**Interfaces:**
- Produces: `fjacquet/ci/actions/mkdocs-publish` — builds mkdocs-material site to `site/` and uploads a Pages artifact. Inputs: `python-version` (default `"3.13"`), `project-install` (`"true"` ⇒ `uv sync`, `"false"` ⇒ lightweight `uvx`; default `"false"`), `uvx-with` (default `"mkdocs-material pymdown-extensions"`). Consumed by `docs-publish` (Task 15).

- [ ] **Step 1: Write `actions/mkdocs-publish/action.yml`**

```yaml
name: mkdocs-publish
description: Build an mkdocs-material site and upload it as a GitHub Pages artifact
inputs:
  python-version:
    default: "3.13"
  project-install:
    description: "true to run 'uv sync' (docs need the package/plugins); false for lightweight uvx"
    default: "false"
  uvx-with:
    description: Space-separated extra packages for uvx mode
    default: "mkdocs-material pymdown-extensions"
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v6
      with:
        python-version: ${{ inputs.python-version }}
    - uses: astral-sh/setup-uv@v8
      with:
        enable-cache: ${{ inputs.project-install }}
    - name: Build (uv sync)
      if: ${{ inputs.project-install == 'true' }}
      shell: bash
      run: |
        uv sync --all-extras --all-groups
        uv run mkdocs build --strict --site-dir site
    - name: Build (uvx)
      if: ${{ inputs.project-install == 'false' }}
      shell: bash
      run: |
        WITH=""
        for p in ${{ inputs.uvx-with }}; do WITH="$WITH --with $p"; done
        uvx $WITH mkdocs build --strict --site-dir site
    - uses: actions/configure-pages@v6
    - uses: actions/upload-pages-artifact@v5
      with:
        path: site
```

- [ ] **Step 2: Validate**

```bash
uvx zizmor --format=plain actions/mkdocs-publish/action.yml
pinact run actions/mkdocs-publish/action.yml && pinact run --check actions/mkdocs-publish/action.yml
```
Expected: zizmor clean; setup-python/setup-uv/configure-pages/upload-pages-artifact pinned.

- [ ] **Step 3: Commit**

```bash
git add actions/mkdocs-publish/action.yml
git commit -m "feat: add mkdocs-publish composite action"
```

---

## Task 8: Reusable workflow — `go-ci`

**Files:**
- Create: `~/Projects/ci/.github/workflows/go-ci.yml`

**Interfaces:**
- Consumes: `actions/setup-go-cache` (Task 4).
- Produces: `fjacquet/ci/.github/workflows/go-ci.yml`. Inputs: `go-version` (string, default `""`), `golangci-version` (string, default `"v2.12.2"`). Optional secret: `CODECOV_TOKEN`. Caller grants `contents: read`.

- [ ] **Step 1: Write `.github/workflows/go-ci.yml`**

```yaml
name: go-ci
on:
  workflow_call:
    inputs:
      go-version:
        type: string
        default: ""
      golangci-version:
        type: string
        default: "v2.12.2"
    secrets:
      CODECOV_TOKEN:
        required: false
permissions:
  contents: read
concurrency:
  group: go-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  quality:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - uses: fjacquet/ci/actions/setup-go-cache@v1
        with:
          go-version: ${{ inputs.go-version }}
      - name: Lint
        uses: golangci/golangci-lint-action@v8
        with:
          version: ${{ inputs.golangci-version }}
          args: --timeout=5m
      - name: Test (race + coverage)
        run: go test -race -coverprofile=coverage.out -covermode=atomic ./...
      - name: Build
        run: go build -v ./...
      - name: Vulncheck
        run: go run golang.org/x/vuln/cmd/govulncheck@latest ./...
      - name: Upload coverage
        uses: codecov/codecov-action@v5
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: coverage.out
          fail_ci_if_error: false
```

- [ ] **Step 2: Validate (SVC)**

Run `actionlint`, `uvx zizmor --format=plain .`, `pinact run` + `pinact run --check`. Expected: all clean. Note: `fjacquet/ci/actions/setup-go-cache@v1` will not resolve until `v1` exists (Task 21) — `actionlint`/`zizmor` only parse it, they do not fetch it, so this is fine. The pilot (Task 17) exercises it against the `task` branch via a temporary ref override.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/go-ci.yml
git commit -m "feat: add go-ci reusable workflow"
```

---

## Task 9: Reusable workflow — `go-security`

**Files:**
- Create: `~/Projects/ci/.github/workflows/go-security.yml`

**Interfaces:**
- Consumes: `actions/setup-go-cache` (Task 4), `actions/sbom` (Task 6), `actions/harden` (Task 3).
- Produces: `go-security.yml`. No inputs. Caller grants `contents: read`. Runs semgrep + SBOM.

- [ ] **Step 1: Write `.github/workflows/go-security.yml`**

```yaml
name: go-security
on:
  workflow_call:
permissions:
  contents: read
concurrency:
  group: go-security-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  semgrep:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - name: Semgrep
        run: semgrep scan --config auto --error --skip-unknown-extensions
  sbom:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: fjacquet/ci/actions/harden@v1
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - uses: fjacquet/ci/actions/setup-go-cache@v1
      - uses: fjacquet/ci/actions/sbom@v1
        with:
          ecosystem: go
```

- [ ] **Step 2: Validate (SVC)**

Expected: clean. (`harden-runner` cannot run inside the semgrep container job, so that job omits it; the `sbom` job runs `harden` first — zizmor accepts this.)

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/go-security.yml
git commit -m "feat: add go-security reusable workflow (semgrep + sbom)"
```

---

## Task 10: Reusable workflow — `go-release`

**Files:**
- Create: `~/Projects/ci/.github/workflows/go-release.yml`

**Interfaces:**
- Consumes: `actions/setup-go-cache` (Task 4).
- Produces: `go-release.yml`. Inputs: `goreleaser-version` (string, default `"~> v2"`), `docker` (boolean, default `true`). Secrets: `HOMEBREW_TAP_GITHUB_TOKEN` (optional). Caller grants `contents: write`, `packages: write`, `id-token: write`.

- [ ] **Step 1: Write `.github/workflows/go-release.yml`**

```yaml
name: go-release
on:
  workflow_call:
    inputs:
      goreleaser-version:
        type: string
        default: "~> v2"
      docker:
        type: boolean
        default: true
    secrets:
      HOMEBREW_TAP_GITHUB_TOKEN:
        required: false
permissions:
  contents: read
jobs:
  goreleaser:
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    permissions:
      contents: write
      packages: write
      id-token: write
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: fjacquet/ci/actions/setup-go-cache@v1
        with:
          cache: "false"
      - name: Set up QEMU
        if: ${{ inputs.docker }}
        uses: docker/setup-qemu-action@v3
      - name: Set up Buildx
        if: ${{ inputs.docker }}
        uses: docker/setup-buildx-action@v3
      - name: Install syft
        uses: anchore/sbom-action/download-syft@v0
      - name: Log in to GHCR
        if: ${{ inputs.docker }}
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: GoReleaser
        uses: goreleaser/goreleaser-action@v7
        with:
          distribution: goreleaser
          version: ${{ inputs.goreleaser-version }}
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          HOMEBREW_TAP_GITHUB_TOKEN: ${{ secrets.HOMEBREW_TAP_GITHUB_TOKEN }}
```

- [ ] **Step 2: Validate (SVC)**

Expected: clean. Note: repos must standardize their `.goreleaser.yaml` to read `HOMEBREW_TAP_GITHUB_TOKEN` (camt-csv currently uses `TAP_GITHUB_TOKEN` — renamed in its pilot task).

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/go-release.yml
git commit -m "feat: add go-release reusable workflow (goreleaser v2)"
```

---

## Task 11: Reusable workflow — `python-ci`

**Files:**
- Create: `~/Projects/ci/.github/workflows/python-ci.yml`

**Interfaces:**
- Consumes: `actions/setup-uv` (Task 5).
- Produces: `python-ci.yml`. Inputs: `python-version` (string, default `"3.12"`), `coverage` (boolean, default `false`). Caller grants `contents: read`.

- [ ] **Step 1: Write `.github/workflows/python-ci.yml`**

```yaml
name: python-ci
on:
  workflow_call:
    inputs:
      python-version:
        type: string
        default: "3.12"
      coverage:
        type: boolean
        default: false
permissions:
  contents: read
concurrency:
  group: python-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  quality:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: fjacquet/ci/actions/setup-uv@v1
        with:
          python-version: ${{ inputs.python-version }}
      - name: Sync
        run: uv sync --all-extras --all-groups
      - name: Ruff lint
        run: uv run ruff check .
      - name: Ruff format check
        run: uv run ruff format --check .
      - name: Test
        if: ${{ !inputs.coverage }}
        run: uv run pytest
      - name: Test (coverage)
        if: ${{ inputs.coverage }}
        run: uv run pytest --cov --cov-report=term-missing
```

- [ ] **Step 2: Validate (SVC)** — Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/python-ci.yml
git commit -m "feat: add python-ci reusable workflow (uv + ruff + pytest)"
```

---

## Task 12: Reusable workflow — `python-release`

**Files:**
- Create: `~/Projects/ci/.github/workflows/python-release.yml`

**Interfaces:**
- Consumes: `actions/setup-uv` (Task 5).
- Produces: `python-release.yml`. Inputs: `python-version` (string, default `"3.12"`), `environment` (string, default `"pypi"`). Uses PyPI OIDC trusted publishing — caller grants `id-token: write`, `contents: read`.

- [ ] **Step 1: Write `.github/workflows/python-release.yml`**

```yaml
name: python-release
on:
  workflow_call:
    inputs:
      python-version:
        type: string
        default: "3.12"
      environment:
        type: string
        default: "pypi"
permissions:
  contents: read
jobs:
  publish:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    environment: ${{ inputs.environment }}
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - uses: fjacquet/ci/actions/setup-uv@v1
        with:
          python-version: ${{ inputs.python-version }}
          enable-cache: "false"
      - name: Build
        run: uv build
      - name: Publish to PyPI (OIDC)
        run: uv publish --trusted-publishing always
```

- [ ] **Step 2: Validate (SVC)** — Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/python-release.yml
git commit -m "feat: add python-release reusable workflow (uv build + PyPI OIDC)"
```

---

## Task 13: Reusable workflow — `web-ci`

**Files:**
- Create: `~/Projects/ci/.github/workflows/web-ci.yml`

**Interfaces:**
- Produces: `web-ci.yml`. Inputs: `node-version` (string, default `"24"`), `package-manager` (string, default `"npm"`). Caller grants `contents: read`. Runs typecheck/lint/test/build via package.json scripts.

- [ ] **Step 1: Write `.github/workflows/web-ci.yml`**

```yaml
name: web-ci
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: "24"
      package-manager:
        type: string
        default: "npm"
permissions:
  contents: read
concurrency:
  group: web-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - uses: actions/setup-node@v6
        with:
          node-version: ${{ inputs.node-version }}
          cache: ${{ inputs.package-manager }}
      - name: Install
        run: ${{ inputs.package-manager }} ci
      - name: Typecheck
        run: ${{ inputs.package-manager }} run typecheck
      - name: Lint
        run: ${{ inputs.package-manager }} run lint
      - name: Test
        run: ${{ inputs.package-manager }} run test:run
      - name: Build
        run: ${{ inputs.package-manager }} run build
```

- [ ] **Step 2: Validate (SVC)** — Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/web-ci.yml
git commit -m "feat: add web-ci reusable workflow (node + npm scripts)"
```

---

## Task 14: Reusable workflow — `web-deploy`

**Files:**
- Create: `~/Projects/ci/.github/workflows/web-deploy.yml`

**Interfaces:**
- Produces: `web-deploy.yml`. Inputs: `node-version` (string, default `"24"`), `package-manager` (string, default `"npm"`), `build-dir` (string, default `"dist"`). Caller grants `contents: read`, `pages: write`, `id-token: write`. Deploys the built app to GitHub Pages.

- [ ] **Step 1: Write `.github/workflows/web-deploy.yml`**

```yaml
name: web-deploy
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: "24"
      package-manager:
        type: string
        default: "npm"
      build-dir:
        type: string
        default: "dist"
permissions:
  contents: read
concurrency:
  group: pages
  cancel-in-progress: false
jobs:
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - uses: actions/setup-node@v6
        with:
          node-version: ${{ inputs.node-version }}
          cache: ${{ inputs.package-manager }}
      - run: ${{ inputs.package-manager }} ci
      - run: ${{ inputs.package-manager }} run build
      - uses: actions/configure-pages@v6
      - uses: actions/upload-pages-artifact@v5
        with:
          path: ${{ inputs.build-dir }}
  deploy:
    needs: build
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    permissions:
      pages: write
      id-token: write
    steps:
      - id: deployment
        uses: actions/deploy-pages@v5
```

- [ ] **Step 2: Validate (SVC)** — Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/web-deploy.yml
git commit -m "feat: add web-deploy reusable workflow (Pages app)"
```

---

## Task 15: Reusable workflow — `docs-publish`

**Files:**
- Create: `~/Projects/ci/.github/workflows/docs-publish.yml`

**Interfaces:**
- Consumes: `actions/mkdocs-publish` (Task 7).
- Produces: `docs-publish.yml`. Inputs: `python-version` (string, default `"3.13"`), `project-install` (boolean, default `false`), `uvx-with` (string, default `"mkdocs-material pymdown-extensions"`). Caller grants `contents: read`, `pages: write`, `id-token: write`.

- [ ] **Step 1: Write `.github/workflows/docs-publish.yml`**

```yaml
name: docs-publish
on:
  workflow_call:
    inputs:
      python-version:
        type: string
        default: "3.13"
      project-install:
        type: boolean
        default: false
      uvx-with:
        type: string
        default: "mkdocs-material pymdown-extensions"
permissions:
  contents: read
concurrency:
  group: pages
  cancel-in-progress: false
jobs:
  build:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: fjacquet/ci/actions/mkdocs-publish@v1
        with:
          python-version: ${{ inputs.python-version }}
          project-install: ${{ inputs.project-install }}
          uvx-with: ${{ inputs.uvx-with }}
  deploy:
    needs: build
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    permissions:
      pages: write
      id-token: write
    steps:
      - id: deployment
        uses: actions/deploy-pages@v5
```

- [ ] **Step 2: Validate (SVC)** — Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/docs-publish.yml
git commit -m "feat: add docs-publish reusable workflow (mkdocs to Pages)"
```

---

## Task 16: Reusable workflow — `web-py-security`

**Files:**
- Create: `~/Projects/ci/.github/workflows/web-py-security.yml`

**Interfaces:**
- Consumes: `actions/harden` (Task 3), `actions/sbom` (Task 6), `actions/setup-uv` (Task 5).
- Produces: `web-py-security.yml`. Inputs: `language` (string, `javascript-typescript` or `python`, required), `ecosystem` (string, `node` or `python`, required). Caller grants `contents: read`, `security-events: write`, `actions: read`. Runs CodeQL + osv-scanner + SBOM.

- [ ] **Step 1: Write `.github/workflows/web-py-security.yml`**

```yaml
name: web-py-security
on:
  workflow_call:
    inputs:
      language:
        type: string
        required: true
      ecosystem:
        type: string
        required: true
permissions:
  contents: read
concurrency:
  group: web-py-security-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  codeql:
    runs-on: ubuntu-24.04
    timeout-minutes: 20
    permissions:
      contents: read
      security-events: write
      actions: read
    steps:
      - uses: fjacquet/ci/actions/harden@v1
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - uses: github/codeql-action/init@v4
        with:
          languages: ${{ inputs.language }}
          queries: security-extended
      - uses: github/codeql-action/analyze@v4
        with:
          category: "/language:${{ inputs.language }}"
  osv-scan:
    uses: google/osv-scanner-action/.github/workflows/osv-scanner-reusable.yml@v2
    permissions:
      contents: read
      security-events: write
      actions: read
  sbom:
    runs-on: ubuntu-24.04
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: fjacquet/ci/actions/harden@v1
      - uses: actions/checkout@v6
        with:
          persist-credentials: false
      - name: Set up toolchain (python)
        if: ${{ inputs.ecosystem == 'python' }}
        uses: fjacquet/ci/actions/setup-uv@v1
      - name: Sync (python)
        if: ${{ inputs.ecosystem == 'python' }}
        run: uv sync --all-extras --all-groups
      - name: Set up toolchain (node)
        if: ${{ inputs.ecosystem == 'node' }}
        uses: actions/setup-node@v6
        with:
          node-version: "24"
      - uses: fjacquet/ci/actions/sbom@v1
        with:
          ecosystem: ${{ inputs.ecosystem }}
```

- [ ] **Step 2: Validate (SVC)**

Expected: clean. Note: the `osv-scan` job calls a third-party *reusable workflow* — pinact pins it to a SHA like any action.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/web-py-security.yml
git commit -m "feat: add web-py-security reusable workflow (codeql + osv + sbom)"
```

---

## Task 17: Pilot — `pscale_exporter` (Go exporter)

Validates `go-ci`, `go-security`, `go-release`, `docs-publish` end-to-end. Because `v1` does not exist yet, the pilot temporarily references the build branch and we promote to `@v1` in Task 21.

**Files (in `~/Projects/ci`):**
- Modify: tag a pre-release ref so pilots can resolve `fjacquet/ci/...@ci-foundation`.

**Files (in `~/Projects/pscale_exporter`):**
- Replace: `.github/workflows/ci.yml`, `.github/workflows/docs.yml`, `.github/workflows/release.yml`
- Preserve as-is (repo-specific, not yet standardized): none — but record any custom `make` gate not covered by `go-ci` in `~/Projects/ci/docs/AUDIT.md` notes.

**Interfaces:**
- Consumes: all Task 8–16 workflows via a temporary `ci-foundation` tag.

- [ ] **Step 1: Push the build branch and create a temporary ref**

```bash
cd ~/Projects/ci
git push -u origin task-01-audit   # if not already; then merge all task branches into one integration branch
git checkout -b ci-foundation
git push -u origin ci-foundation
git tag ci-foundation-0 && git push origin ci-foundation-0
```
(Use the integration branch that contains Tasks 2–16. The throwaway tag `ci-foundation-0` is what pilots reference; it is deleted in Task 21.)

- [ ] **Step 2: Replace `pscale_exporter` workflows with thin callers**

`~/Projects/pscale_exporter/.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request:
permissions:
  contents: read
jobs:
  ci:
    uses: fjacquet/ci/.github/workflows/go-ci.yml@ci-foundation-0
  security:
    uses: fjacquet/ci/.github/workflows/go-security.yml@ci-foundation-0
```

`~/Projects/pscale_exporter/.github/workflows/docs.yml`:
```yaml
name: Docs
on:
  push:
    branches: [main]
    paths: ["docs/**", "mkdocs.yml", ".github/workflows/docs.yml"]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  docs:
    uses: fjacquet/ci/.github/workflows/docs-publish.yml@ci-foundation-0
    with:
      uvx-with: "mkdocs-material pymdown-extensions"
```

`~/Projects/pscale_exporter/.github/workflows/release.yml`:
```yaml
name: Release
on:
  push:
    tags: ["v*"]
permissions:
  contents: read
jobs:
  release:
    uses: fjacquet/ci/.github/workflows/go-release.yml@ci-foundation-0
    permissions:
      contents: write
      packages: write
      id-token: write
    secrets:
      HOMEBREW_TAP_GITHUB_TOKEN: ${{ secrets.HOMEBREW_TAP_GITHUB_TOKEN }}
```

- [ ] **Step 3: Lint the caller files locally**

```bash
cd ~/Projects/pscale_exporter
actionlint
uvx zizmor --format=plain .github/workflows
```
Expected: clean.

- [ ] **Step 4: Push a branch and trigger CI**

```bash
cd ~/Projects/pscale_exporter
git checkout -b ci/standardize
git add .github/workflows
git commit -m "ci: migrate to fjacquet/ci reusable workflows"
git push -u origin ci/standardize
gh pr create --fill
gh pr checks --watch
```
Expected: `go-ci` and `go-security` jobs complete green. If `go-ci` fails because the repo relied on a `make` gate not in the canonical set (e.g. a custom check), record it in `docs/AUDIT.md` and add it back as a small repo-local job in this PR.

- [ ] **Step 5: Validate docs + release without merging to main**

```bash
# docs: trigger via workflow_dispatch on the branch
gh workflow run docs.yml --ref ci/standardize
gh run watch
```
Expected: docs build job green (deploy job will no-op or be skipped off the default branch — acceptable for the pilot; full deploy is verified post-merge). Do NOT push a real `v*` tag yet (would cut a real release); release-job validation is deferred to Task 21's dry run.

- [ ] **Step 6: Commit the pilot result note**

```bash
cd ~/Projects/ci
# append a "Pilot results: pscale_exporter" subsection to docs/AUDIT.md with the run URL and any deviations
git add docs/AUDIT.md
git commit -m "docs: record pscale_exporter pilot results"
```

---

## Task 18: Pilot — `camt-csv` (Go CLI)

Validates `go-ci`, `go-release` for the CLI archetype, and the `HOMEBREW_TAP_GITHUB_TOKEN` rename.

**Files (in `~/Projects/camt-csv`):**
- Replace: `.github/workflows/go.yml` → `ci.yml` (caller), `.github/workflows/goreleaser.yml` → `release.yml` (caller), `.github/workflows/docs.yml` (caller).
- Modify: `.goreleaser.yaml` — rename the brew tap token env from `TAP_GITHUB_TOKEN` to `HOMEBREW_TAP_GITHUB_TOKEN`.
- Delete: `.github/workflows/go-ossf-slsa3-publish.yml` only if AUDIT.md marks SLSA provenance as out-of-scope for v1; otherwise keep it untouched.

- [ ] **Step 1: Rename the goreleaser tap token**

In `~/Projects/camt-csv/.goreleaser.yaml`, change every `{{ .Env.TAP_GITHUB_TOKEN }}` / `TAP_GITHUB_TOKEN` reference to `HOMEBREW_TAP_GITHUB_TOKEN`. Add the new repo secret:
```bash
cd ~/Projects/camt-csv
gh secret set HOMEBREW_TAP_GITHUB_TOKEN --body "$(gh secret list | grep -q TAP_GITHUB_TOKEN && echo REUSE_EXISTING_PAT)"
# In practice: copy the existing TAP_GITHUB_TOKEN PAT value into HOMEBREW_TAP_GITHUB_TOKEN via the GitHub UI or `gh secret set HOMEBREW_TAP_GITHUB_TOKEN < token.txt`
```

- [ ] **Step 2: Write the caller workflows**

`.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request:
permissions:
  contents: read
jobs:
  ci:
    uses: fjacquet/ci/.github/workflows/go-ci.yml@ci-foundation-0
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

`.github/workflows/release.yml`:
```yaml
name: Release
on:
  push: { tags: ["v*"] }
permissions:
  contents: read
jobs:
  release:
    uses: fjacquet/ci/.github/workflows/go-release.yml@ci-foundation-0
    permissions:
      contents: write
      packages: write
      id-token: write
    secrets:
      HOMEBREW_TAP_GITHUB_TOKEN: ${{ secrets.HOMEBREW_TAP_GITHUB_TOKEN }}
```

`.github/workflows/docs.yml`:
```yaml
name: Docs
on:
  push:
    branches: [main]
    paths: ["docs/**", "mkdocs.yml", ".github/workflows/docs.yml"]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  docs:
    uses: fjacquet/ci/.github/workflows/docs-publish.yml@ci-foundation-0
```

Then delete the superseded files:
```bash
cd ~/Projects/camt-csv
git rm .github/workflows/go.yml .github/workflows/goreleaser.yml
```

- [ ] **Step 3: Lint, push, trigger, watch**

```bash
cd ~/Projects/camt-csv
actionlint && uvx zizmor --format=plain .github/workflows
git checkout -b ci/standardize
git add .github/workflows .goreleaser.yaml
git commit -m "ci: migrate to fjacquet/ci reusable workflows; rename brew tap token"
git push -u origin ci/standardize
gh pr create --fill
gh pr checks --watch
```
Expected: `go-ci` green. Record any deviation (camt-csv uses gosec today, not covered by `go-ci`'s govulncheck; decide in AUDIT.md whether to add a gosec step to `go-security` for CLIs or keep it repo-local).

- [ ] **Step 4: Commit pilot note**

```bash
cd ~/Projects/ci
git add docs/AUDIT.md && git commit -m "docs: record camt-csv pilot results"
```

---

## Task 19: Pilot — `finwiz` (Python)

Validates `python-ci`, `docs-publish` (project-install mode), `web-py-security` (python).

**Files (in `~/Projects/finwiz`):**
- Replace: `quality.yml` → `ci.yml` (caller), `docs.yml` (caller), `osv-scanner.yml` + `supply-chain.yml` → `security.yml` (caller).
- Preserve repo-local (not in canonical `python-ci`): finwiz's custom `make check-unittest-mock`, `make check-file-size`, `make coverage-check`, vulture, pylint-duplication. Keep these as a small repo-local `extra-checks.yml` job so no coverage is lost.

- [ ] **Step 1: Write caller workflows**

`.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
permissions:
  contents: read
jobs:
  ci:
    uses: fjacquet/ci/.github/workflows/python-ci.yml@ci-foundation-0
    with:
      python-version: "3.12"
      coverage: true
```

`.github/workflows/extra-checks.yml` (preserve finwiz-specific gates):
```yaml
name: Extra checks
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
permissions:
  contents: read
concurrency:
  group: extra-checks-${{ github.ref }}
  cancel-in-progress: true
jobs:
  extra:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: fjacquet/ci/actions/setup-uv@v1
        with:
          python-version: "3.12"
      - run: uv sync --all-extras --all-groups
      - run: uvx vulture src/finwiz --min-confidence 80
      - run: uvx pylint --disable=all --enable=duplicate-code --min-similarity-lines=37 --score=no src/finwiz
      - run: make check-unittest-mock
      - run: make check-file-size
```

`.github/workflows/docs.yml`:
```yaml
name: Docs
on:
  push:
    branches: [main]
    paths: ["docs/**", "mkdocs.yml", ".github/workflows/docs.yml"]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  docs:
    uses: fjacquet/ci/.github/workflows/docs-publish.yml@ci-foundation-0
    with:
      python-version: "3.12"
      project-install: true
```

`.github/workflows/security.yml`:
```yaml
name: Security
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
permissions:
  contents: read
  security-events: write
  actions: read
jobs:
  security:
    uses: fjacquet/ci/.github/workflows/web-py-security.yml@ci-foundation-0
    with:
      language: python
      ecosystem: python
```

Delete superseded:
```bash
cd ~/Projects/finwiz
git rm .github/workflows/quality.yml .github/workflows/osv-scanner.yml .github/workflows/supply-chain.yml
```

- [ ] **Step 2: Lint, push, trigger, watch**

```bash
cd ~/Projects/finwiz
actionlint && uvx zizmor --format=plain .github/workflows
git checkout -b ci/standardize
git add .github/workflows
git commit -m "ci: migrate to fjacquet/ci reusable workflows; preserve extra checks"
git push -u origin ci/standardize
gh pr create --fill
gh pr checks --watch
```
Expected: `python-ci`, `extra`, `security` jobs green. CodeQL for Python is new for finwiz — if it surfaces findings, record them in AUDIT.md (do not fix app code in this plan).

- [ ] **Step 3: Commit pilot note**

```bash
cd ~/Projects/ci
git add docs/AUDIT.md && git commit -m "docs: record finwiz pilot results"
```

---

## Task 20: Pilot — `vsizer` (Frontend TS)

Validates `web-ci`, `web-deploy`, `web-py-security` (node).

**Files (in `~/Projects/vsizer`):**
- Replace: `static.yml` → `ci.yml` (web-ci) + `deploy.yml` (web-deploy), `codeql.yml` → `security.yml` (web-py-security).
- Preserve repo-local: `container.yml` (vsizer-specific container build + smoke test + trivy) stays untouched — containers are not a standardized archetype concern for v1.

- [ ] **Step 1: Write caller workflows**

`.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
permissions:
  contents: read
jobs:
  ci:
    uses: fjacquet/ci/.github/workflows/web-ci.yml@ci-foundation-0
    with:
      node-version: "24"
```

`.github/workflows/deploy.yml`:
```yaml
name: Deploy
on:
  push: { branches: [main] }
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  deploy:
    uses: fjacquet/ci/.github/workflows/web-deploy.yml@ci-foundation-0
    with:
      node-version: "24"
      build-dir: dist
```

`.github/workflows/security.yml`:
```yaml
name: Security
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule:
    - cron: "23 4 * * 1"
permissions:
  contents: read
  security-events: write
  actions: read
jobs:
  security:
    uses: fjacquet/ci/.github/workflows/web-py-security.yml@ci-foundation-0
    with:
      language: javascript-typescript
      ecosystem: node
```

Delete superseded:
```bash
cd ~/Projects/vsizer
git rm .github/workflows/static.yml .github/workflows/codeql.yml
```

- [ ] **Step 2: Lint, push, trigger, watch**

```bash
cd ~/Projects/vsizer
actionlint && uvx zizmor --format=plain .github/workflows
git checkout -b ci/standardize
git add .github/workflows
git commit -m "ci: migrate to fjacquet/ci reusable workflows"
git push -u origin ci/standardize
gh pr create --fill
gh pr checks --watch
```
Expected: `web-ci`, `security` green; `web-deploy` build job green (deploy runs post-merge on main). vsizer's `static.yml` had extra gates (npm audit, osv SARIF gate, SBOM component check) — record in AUDIT.md whether to fold `npm audit` into `web-ci` as an input or keep repo-local.

- [ ] **Step 3: Commit pilot note**

```bash
cd ~/Projects/ci
git add docs/AUDIT.md && git commit -m "docs: record vsizer pilot results"
```

---

## Task 21: Cut `v1`, finalize README, dry-run release

**Files:**
- Modify: `~/Projects/ci/README.md` (complete the per-workflow usage table from pilot learnings)

**Interfaces:**
- Produces: the `v1` tag + `v1` major moving tag that all callers reference.

- [ ] **Step 1: Fold pilot learnings into the workflows**

Apply any interface refinements discovered in Tasks 17–20 (e.g. an added input, a preserved gate). Re-run the SVC on `~/Projects/ci`. Expected: clean.

- [ ] **Step 2: Complete `README.md`**

Fill the 9-row usage table: for each workflow, the caller snippet, required `permissions`, inputs with defaults, and secrets. Use the exact caller YAML proven in the pilots.

- [ ] **Step 3: Merge integration branch to `main` and tag**

```bash
cd ~/Projects/ci
git checkout main
git merge --no-ff ci-foundation
git push origin main
git tag -a v1.0.0 -m "fjacquet/ci v1.0.0"
git tag -f v1 v1.0.0
git push origin v1.0.0
git push -f origin v1
```

- [ ] **Step 4: Repoint pilot callers from `@ci-foundation-0` to `@v1`**

In each of the 4 pilot repos' caller workflows, replace `@ci-foundation-0` with `@v1`, commit, push, and confirm `gh pr checks --watch` stays green. Then merge the 4 pilot PRs.

- [ ] **Step 5: Release dry-run on one Go pilot**

```bash
cd ~/Projects/pscale_exporter
# verify goreleaser config is valid without publishing
go run github.com/goreleaser/goreleaser/v2@latest release --snapshot --clean --skip=publish
```
Expected: a local `dist/` build with archives + SBOM, no publish. Confirms `go-release` will work on a real tag.

- [ ] **Step 6: Delete the throwaway ref and commit final docs**

```bash
cd ~/Projects/ci
git tag -d ci-foundation-0
git push origin :refs/tags/ci-foundation-0
git push origin :refs/heads/ci-foundation
git add README.md docs/AUDIT.md
git commit -m "docs: finalize v1 usage guide and pilot results"
git push origin main
```

---

## Self-Review

**Spec coverage (DESIGN.md → tasks):**
- D1 scope / inventory → Task 1.
- D2 hybrid mechanism → composite actions (Tasks 3–7) + reusable workflows (Tasks 8–16).
- D3 SHA-pin + Dependabot → SVC `pinact` everywhere + Task 2 `dependabot.yml`; callers use `@v1` (Task 21).
- D4 best-practice defaults → Global Constraints, applied in every workflow job.
- D5 best-of-breed security → `go-security` (Task 9, semgrep+sbom) and `web-py-security` (Task 16, codeql+osv+sbom).
- D6 `fjacquet/ci` → Task 2.
- D7 mkdocs standard → `docs-publish` (Task 15) + `mkdocs-publish` (Task 7). (Jekyll migration is Phase 4 / next plan — out of scope here, noted.)
- 9 reusable workflows → Tasks 8–16 (one each). 5 composite actions → Tasks 3–7 (one each). Pilots → Tasks 17–20. `v1` → Task 21.

**Deferred to the Phase 3–4 plan (intentionally out of scope):** fleet rollout to the remaining ~36 repos, Jekyll→mkdocs migration (`para-files`, `pdf2md`), sonar retirement, the npm-release decision (`vgpu-advisor`, `brave-search-mcp-server`), SLSA provenance decision (`camt-csv`).

**Placeholder scan:** none. Where a value is intentionally repo-discovered (e.g. AUDIT classification, preserved make gates), the step gives the exact command to resolve it.

**Type/name consistency:** composite action paths (`fjacquet/ci/actions/<name>@v1`) and workflow filenames are consistent between their defining task's Interfaces block and every consuming workflow. Secret name `HOMEBREW_TAP_GITHUB_TOKEN` is used consistently in `go-release` (Task 10) and both Go pilots (Tasks 17, 18). Input names (`project-install`, `uvx-with`, `ecosystem`, `language`, `package-manager`, `build-dir`) match between definition and use.
