# Fleet Health Audit + Refresh — Design Spec

- **Date:** 2026-06-20
- **Status:** Approved (design) — pending implementation plan
- **Owner:** Fred (fjacquet)
- **Ledger:** `~/Projects/ci/.superpowers/sdd/progress.md`

## Goal

A fleet-wide health pass over every owned, in-scope repo (all now on `fjacquet/ci@v1`),
auditing five dimensions and **fixing** what's stale — a victory-lap consolidation after
the CI/CD standardization rollout.

## Scope

- **In:** the ~36 owned in-scope repos classified in `~/Projects/ci/docs/AUDIT.md`
  (Go exporters & CLIs, Python apps/libs, frontend apps, npm/MCP libs).
- **Out:** `code-review-graph` (remote = tirth8205, not owned); `ppdm2jira`
  (PowerShell, no git remote, no archetype); `fjacquet/ci` itself is audited lightly
  (docs only) but never modified by audit subagents.

## Mode (decided)

- **Full:** audit → fix docs/CLAUDE.md/ADRs via PRs → **cut releases**.
- **Versioning:** **patch-only, conservative** — always `vX.Y.(Z+1)` from the latest tag,
  regardless of commit content. Never auto-signals minor/major.
- **Execution:** one **Sonnet** subagent per repo, in waves of ~6–8 concurrent.
  **Controller (main, Opus)** consolidates, merges PRs, and pushes release tags.

## Audit dimensions → per-repo scorecard

Each repo gets a rating per dimension and an action:

1. **CLAUDE.md** — `GOOD` / `STALE` (→ fix) / `MISSING` (→ create a minimal, accurate one:
   what the repo is, archetype, `make`/npm commands, `@v1` CI, key paths).
2. **ADRs** (`docs/adr/`) — `GOOD` / `STALE` (→ fix obvious drift) / `NONE` (note; do NOT
   invent decisions).
3. **Docs** (README + mkdocs site) — `GOOD` / `STALE` (→ fix to match current code).
   **mkdocs requirement:** `mkdocs.yml` MUST set `repo_url: https://github.com/fjacquet/<repo>`
   + `repo_name: fjacquet/<repo>`, AND surface the repo version via mkdocs-material
   `extra.version`. Version value: the **proposed patch tag** for repos that will be released
   this pass (so the docs ship matching the about-to-be-cut release), else the **current latest
   tag**. README badges/links to the repo + latest release where appropriate.
4. **Pending merges** — list OPEN PRs + local/remote branches ahead of default (report only;
   do not merge others' WIP).
5. **Release** — unreleased commits since the latest tag **AND** a release mechanism present
   (goreleaser/`release.yml`, npm publish, PyPI)? → propose the **patch** version.

## Fix policy (full mode)

- All doc/CLAUDE.md/ADR/mkdocs fixes for a repo go in **ONE PR** per repo:
  branch `chore/health-refresh`, title `chore: fleet health refresh`. Subagent drives CI green, STOPS.
- **Controller** merges each refresh PR (squash, `--merge` fallback; repoint stale required
  checks as needed — same as rollout).
- **Release** fires only for repos with a real release mechanism + genuine unreleased changes.
  Sequencing: cut the patch tag **after** that repo's refresh PR merges, so the release
  includes the refresh (and the mkdocs version matches). Controller pushes the tag.
  Repos with no release mechanism → refreshed but NOT tagged (noted in dashboard).

## Per-repo subagent contract (Sonnet)

INPUT: repo name, local path, GitHub name, live default branch (controller pre-fetches),
archetype.
DOES (read-mostly + one PR):
1. Audit all 5 dimensions against the actual code; produce the scorecard.
2. Apply fixes (CLAUDE.md/ADR/docs/mkdocs) on `chore/health-refresh`; for `MISSING` CLAUDE.md,
   create a minimal accurate one.
3. Open the PR, drive the repo's CI green (≤3 repo-side iters), STOP. Never edit `fjacquet/ci`.
4. Compute the proposed patch version + whether a release is warranted (has mechanism + unreleased changes).
RETURNS (structured): scorecard (5 ratings), PR #/url + CI state, list of other open PRs/branches,
release recommendation (current tag → proposed patch tag, mechanism), notable findings.

## Controller flow

1. Pre-flight: live default branch + latest tag + release-mechanism detection per repo.
2. Dispatch waves of Sonnet subagents.
3. As each reports green: merge the refresh PR (repoint protection if needed).
4. After a repo's PR merges, if a release is warranted: push the patch tag (`git tag vX.Y.Z+1 <main HEAD>` + push).
5. Build the consolidated dashboard.

## Deliverable

`~/Projects/ci/docs/FLEET-HEALTH-2026-06-20.md` — table per repo: CLAUDE.md | ADR | docs |
open-PRs | release (old→new tag or "n/a") | PR# | notes. Plus the real PRs merged and tags pushed.

## Guardrails

- Subagents NEVER merge, push tags, modify branch protection, or edit `fjacquet/ci` — controller only.
- Releases are irreversible: only patch bumps; only repos with a real publish mechanism; only after refresh merges.
- Writes to docs/CLAUDE.md/ADRs only — no application-code changes in the refresh PRs.

## Out of scope

- Minor/major version bumps; changelog authoring beyond what release tooling generates.
- New ADRs for decisions that don't exist; net-new feature docs.
- Auditing/modifying `code-review-graph`, `ppdm2jira`.
