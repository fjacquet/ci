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

| Workflow | File | Description | Required caller permissions |
|----------|------|-------------|------------------------------|
| self-check | `.github/workflows/self-check.yml` | Runs actionlint + zizmor + pinact on this repo | `contents: read` |
| ci-node | _coming in Task 8_ | Node.js lint, test, build | `contents: read` |
| ci-python | _coming in Task 9_ | Python lint, test, coverage | `contents: read` |
| ci-go | _coming in Task 10_ | Go vet, test, build | `contents: read` |
| ci-docker | _coming in Task 11_ | Docker build + push to GHCR | `contents: read`, `packages: write` |
| release | _coming in Task 12_ | Semantic release + changelog | `contents: write`, `id-token: write` |
| codeql | _coming in Task 13_ | CodeQL SAST scan | `contents: read`, `security-events: write` |
| osv-scan | _coming in Task 14_ | OSV dependency scan | `contents: read`, `security-events: write` |
| pages | _coming in Task 15_ | Deploy static site to GitHub Pages | `contents: read`, `pages: write`, `id-token: write` |

## Usage example

```yaml
# .github/workflows/ci.yml  (in a caller repo)
jobs:
  ci:
    uses: fjacquet/ci/.github/workflows/ci-node.yml@v1
    permissions:
      contents: read
    with:
      node-version: "20"
```

Replace `ci-node.yml` with whichever workflow you need and supply its `with:` inputs as documented in the workflow file itself.

## Self-check

The [`self-check`](.github/workflows/self-check.yml) workflow validates this repo on every push/PR:

1. **actionlint** — lints all workflow YAML for syntax and semantic errors.
2. **zizmor** — audits for supply-chain and security issues; allowed orgs defined in `zizmor.yml`.
3. **pinact check** — ensures every third-party action is pinned to a full commit SHA.
