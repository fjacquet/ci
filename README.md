# fjacquet/ci

Central repository of reusable GitHub Actions workflows and composite actions for the `fjacquet` organisation.

## Versioning policy

- Callers pin to a **major tag** (e.g. `@v1`). Breaking changes bump to `@v2`.
- Every action reference inside these workflows is SHA-pinned and managed by [pinact](https://github.com/suzuki-shunsuke/pinact).
- [Dependabot](/.github/dependabot.yml) opens weekly PRs to keep SHAs current.

## Security policy

- All actions must come from orgs listed in [`zizmor.yml`](/zizmor.yml) (`actions/*`, `astral-sh/*`, `step-security/*`, `fjacquet/*`, etc.).
- `step-security/harden-runner` with `egress-policy: audit` is required on every job.
- `zizmor` and `actionlint` run in CI on every PR via the [self-check](.github/workflows/self-check.yml) workflow.

## Workflows

| Workflow | File | Purpose | Required caller permissions | Optional secrets |
|----------|------|---------|------------------------------|-----------------|
| go-ci | `.github/workflows/go-ci.yml` | Go lint, test, build, SBOM, coverage upload | `contents: read` | `CODECOV_TOKEN` |
| go-security | `.github/workflows/go-security.yml` | Go semgrep security scan | `contents: read` | — |
| go-release | `.github/workflows/go-release.yml` | GoReleaser cross-platform release + GHCR push | `contents: write`, `packages: write`, `id-token: write` | `HOMEBREW_TAP_GITHUB_TOKEN` |
| python-ci | `.github/workflows/python-ci.yml` | Python lint, test, build, SBOM, coverage upload | `contents: read` | `CODECOV_TOKEN` |
| python-security | `.github/workflows/python-security.yml` | Python semgrep + OSV vulnerability scan | `contents: read` | — |
| python-release | `.github/workflows/python-release.yml` | uv build + PyPI trusted publishing | `contents: read`, `id-token: write` | — |
| python-app-release | `.github/workflows/python-app-release.yml` | Python app release: wheel/sdist + SBOM + GitHub Release + optional GHCR image (no PyPI) | `contents: write`, `packages: write` | — |
| web-ci | `.github/workflows/web-ci.yml` | Node.js typecheck, lint, test, build | `contents: read` | — |
| web-deploy | `.github/workflows/web-deploy.yml` | Node.js build + deploy to GitHub Pages | `contents: read` (build job), `pages: write`, `id-token: write` (deploy job) | — |
| web-security | `.github/workflows/web-security.yml` | CodeQL SAST + OSV scan + SBOM for JS/TS | `contents: read`, `security-events: write`, `actions: read` | — |
| docs-publish | `.github/workflows/docs-publish.yml` | MkDocs build + deploy to GitHub Pages | `contents: read` (build job), `pages: write`, `id-token: write` (deploy job) | — |

## Consumer requirements

### Go repos

Must expose the canonical Makefile target set from [`templates/Makefile.go`](templates/Makefile.go):
`all`, `clean`, `install`, `tools`, `lint`, `format`, `test`, `build`, `vuln`, `sbom`, `security`, `docs`, `coverage-upload`, `release`, `ci`.

The `tools` target installs `golangci-lint`, `govulncheck`, and `goreleaser` via `go install`.

Copy the four caller workflows from [`templates/workflows/`](templates/workflows/) into the
consumer's `.github/workflows/` — `ci.yml`, `security.yml`, `release.yml`, `docs.yml`. They are
thin callers of the `go-*` / `docs-publish` reusable workflows above; keep them thin (no inlined
build steps). The consumer still owns `.goreleaser.yaml`, `Dockerfile.goreleaser`, the MkDocs
site, and a `gomod` + `docker` Dependabot config — but **not** a `github-actions` Dependabot
ecosystem, since the pinned actions now live in this repo.

### Python repos

Must expose the canonical Makefile target set from [`templates/Makefile.python`](templates/Makefile.python).
Dev dependencies must include `cyclonedx-py` (for `make sbom`) and `mkdocs-material` (for `make docs`).

### Frontend (web) repos

Stay npm-native — no Makefile required. Scripts `typecheck`, `lint`, `test:run`, and `build` must be defined in `package.json`.

## Usage example

```yaml
# .github/workflows/ci.yml  (in a caller repo)
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: fjacquet/ci/.github/workflows/go-ci.yml@v1
    permissions:
      contents: read
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

Replace `go-ci.yml` with whichever workflow you need and supply its `with:` inputs and `secrets:` as documented in the workflow file itself.

## Self-check

The [`self-check`](.github/workflows/self-check.yml) workflow validates this repo on every push/PR:

1. **actionlint** — lints all workflow YAML for syntax and semantic errors.
2. **zizmor** — audits for supply-chain and security issues; allowed orgs defined in `zizmor.yml`.
3. **pinact check** — ensures every third-party action is pinned to a full commit SHA.
