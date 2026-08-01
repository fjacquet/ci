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
| web-deploy | `.github/workflows/web-deploy.yml` | Node.js build + deploy to GitHub Pages (`build-script` selects the npm script) | `contents: read` (build job), `pages: write`, `id-token: write` (deploy job) | — |
| web-security | `.github/workflows/web-security.yml` | CodeQL SAST + OSV scan + SBOM for JS/TS | `contents: read`, `security-events: write`, `actions: read` | — |
| npm-release | `.github/workflows/npm-release.yml` | npm publish to npmjs.org via trusted publishing (OIDC) + provenance + GitHub Release | `contents: write`, `id-token: write` | — |
| docs-publish | `.github/workflows/docs-publish.yml` | MkDocs build + deploy to GitHub Pages (exports `SITE_FOOTER`, see [Docs sites: the version source link](#docs-sites-the-version-source-link)) | `contents: read` (build job), `pages: write`, `id-token: write` (deploy job) | — |

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

### Docs sites: the version source link

`docs-publish` exports one environment variable to `make docs`:

| Variable | Shape | Example |
|----------|-------|---------|
| `SITE_FOOTER` | A complete, single-line HTML fragment containing one anchor. Never empty. | `Source: <a href="https://github.com/fjacquet/obs_exporter/tree/v3.4.0" rel="noopener">fjacquet/obs_exporter v3.4.0</a>` |

The link target is `<server_url>/<repository>/tree/<version>`, where `<version>` is, in order
of preference: the pushed tag name, the nearest tag reachable from `HEAD`
(`git describe --tags --abbrev=0`), or the short commit SHA. A version containing anything
outside `[A-Za-z0-9._/-]` falls back to the short SHA, so the exported HTML is always
composed here and never interpolates untrusted text.

To render it, a consumer makes **one** change to `mkdocs.yml`:

```yaml
copyright: !ENV [SITE_FOOTER, ""]
```

mkdocs-material renders `copyright` as raw HTML in the site footer, and MkDocs' `!ENV`
tag supplies the default when the variable is unset — so local `mkdocs serve` and
`make docs` outside CI keep working with an empty footer instead of failing.

Two related rules for consumer `mkdocs.yml`:

- **Delete any `extra.version:` key.** It renders nothing (mkdocs-material only shows a
  version selector with `extra.version.provider: mike` plus a theme override directory)
  and it goes stale silently. `SITE_FOOTER` replaces it.
- **A repo that already sets a literal `copyright:`** (e.g. `MIT Licensed`) must fold that
  text into the `!ENV` default, not keep a second key:
  `copyright: !ENV [SITE_FOOTER, "MIT Licensed"]`.

Adopting this is optional per repo: a consumer that has not added the `copyright:` line
builds exactly as before.

### Published npm packages

`npm-release` publishes to npmjs.org with **trusted publishing** — there is no `NPM_TOKEN` anywhere.
Authentication is an OIDC exchange, and npm generates a provenance attestation automatically.

Two things are easy to get wrong:

- **The trusted publisher on npmjs.com must name the *caller* workflow file** (e.g. `release.yml` in
  your package repo), not `npm-release.yml`. When `workflow_call` is involved, npm validates the
  calling workflow, not the one that actually runs `npm publish`.
- **`id-token: write` must be declared in the caller too**, not only here. The parent workflow has to
  be allowed to mint the OIDC token in the first place.

The caller must also set an `environment` (default `npm`) that exists in the repo settings, and
`package.json` needs a `repository` field matching the GitHub URL. The workflow refuses to publish
when the git tag does not match `v<package.json version>`, so a mismatched tag fails loudly instead
of burning a version number on the registry.

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
