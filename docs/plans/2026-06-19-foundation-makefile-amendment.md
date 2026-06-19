# Foundation Plan — Makefile Amendment (supersedes Tasks 8–16)

> Amends `2026-06-19-foundation.md` per **D8** (DESIGN.md). Go + Python build is
> driven entirely through a canonical Makefile; the reusable workflows call
> `make <target>`. Frontend stays npm-native. Tasks 1–7 are unchanged and
> already complete. This amendment re-specifies Tasks 8–16 and adds Task M
> (Makefile templates). The Global Constraints and Standard Validation Cycle
> (SVC) from the base plan still apply unchanged.

## Task M (NEW, do first): canonical Makefile templates

**Files:** Create `templates/Makefile.go`, `templates/Makefile.python` in `~/Projects/ci`.

These are the standard target set every Go/Python repo adopts. Workflows call these target names; repo authors may extend internals but MUST keep the names.

### `templates/Makefile.go`
```makefile
# Canonical Go Makefile — fjacquet/ci standard interface (do not rename targets)
.DEFAULT_GOAL := all
DIST  ?= dist
COVER ?= coverage.out
GOLANGCI_VERSION ?= v2.12.2

.PHONY: all clean install tools lint format test build vuln sbom security docs coverage-upload release ci

all: clean lint test build

clean:
	rm -rf $(DIST) site $(COVER) *.sarif

install:
	go mod download

tools:
	go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@$(GOLANGCI_VERSION)
	go install golang.org/x/vuln/cmd/govulncheck@latest

lint:
	golangci-lint run --timeout=5m

format:
	golangci-lint fmt

test:
	go test -race -coverprofile=$(COVER) -covermode=atomic ./...

build:
	go build -v ./...

vuln:
	go run golang.org/x/vuln/cmd/govulncheck@latest ./...

sbom:
	mkdir -p $(DIST)
	go run github.com/CycloneDX/cyclonedx-gomod/cmd/cyclonedx-gomod@latest mod -json -output $(DIST)/sbom.cdx.json

security:
	uvx semgrep scan --config auto --error --skip-unknown-extensions

docs:
	uvx --with mkdocs-material --with pymdown-extensions mkdocs build --strict --site-dir site

coverage-upload:
	uvx codecov-cli upload-process --file $(COVER) || true

release:
	goreleaser release --clean

ci: lint test build vuln
```

### `templates/Makefile.python`
```makefile
# Canonical Python Makefile — fjacquet/ci standard interface (do not rename targets)
.DEFAULT_GOAL := all
DIST ?= dist

.PHONY: all clean install tools lint format test build vuln sbom security docs coverage-upload release ci

all: clean lint test build

clean:
	rm -rf $(DIST) site .coverage coverage.xml *.sarif

install:
	uv sync --all-extras --all-groups

tools: install

lint:
	uv run ruff check .
	uv run ruff format --check .

format:
	uv run ruff format .

test:
	uv run pytest --cov --cov-report=xml --cov-report=term-missing

build:
	uv build

vuln:
	uvx osv-scanner scan --lockfile=uv.lock || true

sbom:
	mkdir -p $(DIST)
	uv run cyclonedx-py environment --output-format JSON --output-file $(DIST)/sbom.cdx.json

security:
	uvx semgrep scan --config auto --error --skip-unknown-extensions

docs:
	uv run mkdocs build --strict --site-dir site

coverage-upload:
	uvx codecov-cli upload-process --file coverage.xml || true

release:
	uv build
	uv publish --trusted-publishing always

ci: lint test build
```

**Validate:** `make -n -f templates/Makefile.go all` and `make -n -f templates/Makefile.python all` (dry-run; expect the target chain to expand without error). Commit: `feat: add canonical Go and Python Makefile templates`.

---

## Revised composite action: `mkdocs-publish`

Change the build to use `make docs` (the Makefile encapsulates uvx-vs-uv-sync per archetype). Replace the two "Build (...)" steps with a single step and keep the Pages upload:
```yaml
    - name: Build docs
      shell: bash
      run: make docs
    - uses: actions/configure-pages@<pinned>
    - uses: actions/upload-pages-artifact@<pinned>
      with:
        path: site
```
Keep `setup-python` + `setup-uv` steps (needed so `make docs` has uv). Drop the `project-install`/`uvx-with` inputs (now the Makefile's `docs` target owns that). Re-run zizmor + `pinact run --check`. Commit: `refactor(mkdocs-publish): build via make docs`.

---

## Revised reusable workflows (make-based). Tasks 8–16.

All workflows keep the per-job best-practice defaults from the base plan
(harden-runner first, checkout `persist-credentials: false`, least-priv
`permissions`, `concurrency`, `timeout-minutes`, `ubuntu-24.04`). First-party
`fjacquet/ci/...@v1` refs stay tag-pinned; third-party actions SHA-pinned via
`pinact`. Commit each workflow separately.

### Task 8 — `go-ci.yml`
```yaml
name: go-ci
on:
  workflow_call:
    inputs:
      go-version: { type: string, default: "" }
    secrets:
      CODECOV_TOKEN: { required: false }
permissions:
  contents: read
concurrency:
  group: go-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  ci:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with: { egress-policy: audit }
      - uses: actions/checkout@v6
        with: { persist-credentials: false }
      - uses: fjacquet/ci/actions/setup-go-cache@v1
        with: { go-version: "${{ inputs.go-version }}" }
      - uses: astral-sh/setup-uv@v8   # for uvx semgrep/codecov in make targets
      - run: make tools
      - run: make ci
      - run: make sbom
      - run: make coverage-upload
        env:
          CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

### Task 9 — `go-security.yml`
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
  security:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with: { egress-policy: audit }
      - uses: actions/checkout@v6
        with: { persist-credentials: false }
      - uses: astral-sh/setup-uv@v8
      - run: make security
```

### Task 10 — `go-release.yml`
Docker setup stays as actions (runner infra); the release logic is `make release`.
```yaml
name: go-release
on:
  workflow_call:
    secrets:
      HOMEBREW_TAP_GITHUB_TOKEN: { required: false }
permissions:
  contents: read
jobs:
  release:
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    permissions:
      contents: write
      packages: write
      id-token: write
    steps:
      - uses: step-security/harden-runner@v2
        with: { egress-policy: audit }
      - uses: actions/checkout@v6
        with: { fetch-depth: 0, persist-credentials: false }
      - uses: fjacquet/ci/actions/setup-go-cache@v1
        with: { cache: "false" }
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: anchore/sbom-action/download-syft@v0
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - run: make release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          HOMEBREW_TAP_GITHUB_TOKEN: ${{ secrets.HOMEBREW_TAP_GITHUB_TOKEN }}
```

### Task 11 — `python-ci.yml`
```yaml
name: python-ci
on:
  workflow_call:
    inputs:
      python-version: { type: string, default: "3.12" }
    secrets:
      CODECOV_TOKEN: { required: false }
permissions:
  contents: read
concurrency:
  group: python-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  ci:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with: { egress-policy: audit }
      - uses: actions/checkout@v6
        with: { fetch-depth: 0, persist-credentials: false }
      - uses: fjacquet/ci/actions/setup-uv@v1
        with: { python-version: "${{ inputs.python-version }}" }
      - run: make install
      - run: make ci
      - run: make sbom
      - run: make coverage-upload
        env:
          CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

### Task 12 — `python-release.yml`
```yaml
name: python-release
on:
  workflow_call:
    inputs:
      python-version: { type: string, default: "3.12" }
      environment: { type: string, default: "pypi" }
permissions:
  contents: read
jobs:
  release:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    environment: ${{ inputs.environment }}
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: step-security/harden-runner@v2
        with: { egress-policy: audit }
      - uses: actions/checkout@v6
        with: { persist-credentials: false }
      - uses: fjacquet/ci/actions/setup-uv@v1
        with: { python-version: "${{ inputs.python-version }}", enable-cache: "false" }
      - run: make release
```

### Task 13 — `python-security.yml` (NEW; replaces python's slice of old web-py-security)
```yaml
name: python-security
on:
  workflow_call:
    inputs:
      python-version: { type: string, default: "3.12" }
permissions:
  contents: read
concurrency:
  group: python-security-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  security:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
    steps:
      - uses: step-security/harden-runner@v2
        with: { egress-policy: audit }
      - uses: actions/checkout@v6
        with: { persist-credentials: false }
      - uses: fjacquet/ci/actions/setup-uv@v1
        with: { python-version: "${{ inputs.python-version }}" }
      - run: make install
      - run: make security
      - run: make vuln
```

### Task 14 — `web-ci.yml` (npm-native; unchanged from base plan Task 13)
Use the base plan's `web-ci.yml` verbatim (node + npm scripts: install/typecheck/lint/test/build).

### Task 15 — `web-deploy.yml` (npm-native; unchanged from base plan Task 14)
Use the base plan's `web-deploy.yml` verbatim (npm build → Pages).

### Task 16 — `docs-publish.yml` + `web-security.yml`

`docs-publish.yml` — unchanged from base plan Task 15 EXCEPT it now relies on the revised `mkdocs-publish` composite (which calls `make docs`). Drop the `project-install`/`uvx-with` inputs from the caller surface; keep `python-version`.

`web-security.yml` (renamed from `web-py-security`, frontend-only, action-based):
```yaml
name: web-security
on:
  workflow_call:
permissions:
  contents: read
concurrency:
  group: web-security-${{ github.workflow }}-${{ github.ref }}
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
      - uses: step-security/harden-runner@v2
        with: { egress-policy: audit }
      - uses: actions/checkout@v6
        with: { persist-credentials: false }
      - uses: github/codeql-action/init@v4
        with: { languages: javascript-typescript, queries: security-extended }
      - uses: github/codeql-action/analyze@v4
        with: { category: "/language:javascript-typescript" }
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
        with: { persist-credentials: false }
      - uses: actions/setup-node@v6
        with: { node-version: "24" }
      - uses: fjacquet/ci/actions/sbom@v1
        with: { ecosystem: node }
```

---

## Pilot updates (Tasks 17–20)

- **Go/Python pilots (`pscale_exporter`, `camt-csv`, `finwiz`):** before migrating the caller workflows, ensure the repo has a Makefile exposing the canonical target set. `pscale_exporter` and `finwiz` already have rich Makefiles — reconcile their targets to the canonical names (add any missing: `all`, `clean`, `ci`, `sbom`, `security`, `docs`, `coverage-upload`, `release`). `camt-csv` likewise. Use `templates/Makefile.<lang>` as the reference. Record reconciliation in AUDIT.md.
- **Frontend pilot (`vsizer`):** unchanged — npm-native, no Makefile.
- Caller workflows: Go/Py pilots call `go-ci`/`python-ci`/`*-security`/`*-release`/`docs-publish`; `vsizer` calls `web-ci`/`web-deploy`/`web-security`.

## Net workflow inventory (now 10)
`go-ci`, `go-security`, `go-release`, `python-ci`, `python-security`, `python-release`, `web-ci`, `web-deploy`, `web-security`, `docs-publish`.

## Self-review note
The `coverage-upload` and `security` make targets shell out to `uvx codecov-cli`/`uvx semgrep`/`uvx osv-scanner`; the workflows install uv (via `setup-uv` or `astral-sh/setup-uv`) so `uvx` is present even in Go jobs. Pilots will confirm these resolve on the runner; any gap is recorded and fixed before `v1`.
