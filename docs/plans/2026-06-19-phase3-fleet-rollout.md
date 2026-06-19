# Phase 3 — Fleet Rollout Plan

> Rolls the **validated** `fjacquet/ci@v1` standard onto the remaining ~36 owned
> repos. The migration recipe is proven (4 pilots green, zero central bugs). This
> plan is a rollout, not new design: it carries the per-archetype recipe, the repo
> lists, the special cases, and the execution model. Source of truth for workflow
> content: the 4 pilot PRs (#20 pscale, #13 camt-csv, #73 finwiz, #38 vsizer) and
> `docs/plans/2026-06-19-foundation-makefile-amendment.md`.

## Decisions (locked)
- **Include** `idrac_exporter` (go-exporter) and `Nano-Banana-MCP` (npm/node, ci+security only).
- **Auto-merge when green:** each migration PR gets `gh pr merge --auto --squash`; it lands once `go-ci`/`python-ci`/`web-ci` + security checks pass. Report merged vs. failed-needs-attention.
- **npm-publishing repos** (`vgpu-advisor`, `Nano-Banana-MCP`): migrate **ci + security only**; leave their existing npm-publish workflow untouched. No `web-release` workflow this phase.

## The migration recipe (per archetype — proven by pilots)

**Go (exporter + cli):** reconcile/create `Makefile` to the canonical target set (`templates/Makefile.go`: `tools` MUST install goreleaser; add `security`=uvx semgrep, `coverage-upload`, `docs`; `test` emits coverage). Replace `ci.yml`/`docs.yml`/`release.yml` with thin callers → `go-ci`/`docs-publish`/`go-release@v1` + `go-security@v1`. **Older-Go repos:** if `make tools` fails (golangci-lint/goreleaser need Go 1.25), override `GOLANGCI_VERSION`/`GORELEASER_VERSION` in the repo Makefile to Go-1.24-compatible (`v2.8.0`/`v2.7.0`) — as camt-csv did. Pass `CODECOV_TOKEN` secret if the repo has it.

**Python:** reconcile/create `Makefile` to `templates/Makefile.python` (preserve repo-specific gates in a repo-local `extra-checks.yml`). Callers → `python-ci`/`python-security`/`docs-publish@v1`. Ensure `cyclonedx-bom` (provides `cyclonedx-py`) + `mkdocs-material` are dev-deps. `lint`/`format` use `uv run ruff` (not bare). Suppress pre-existing semgrep false-positives with documented `# nosemgrep` (as finwiz did). `python-release@v1` only for PyPI publishers.

**Frontend (npm-native):** callers → `web-ci`/`web-deploy`/`web-security@v1` (node 24, npm). Verify `package.json` has `typecheck`/`lint`/`test:run`/`build` scripts (note mismatches). Delete superseded `static.yml`/`codeql.yml`; keep repo-specific extras (`container.yml`, Claude-review workflows). Write caller files via Bash heredoc (semgrep hook false-positives on first-party `@v1` refs).

**Per-repo Definition of Done:** PR open, `*-ci` + `*-security` checks green, auto-merge enabled. (Release = tag-triggered, not gated here; docs deploy = post-merge on main.)

## Batch 1 — Go exporters (10 repos)
`cee-exporter`, `ecs_exporter`, `idrac_exporter`, `nbu_exporter`, `nsr_exporter`, `pflex_exporter`, `pmax_exporter`, `ppdd_exporter`, `ppdm_exporter`, `pstore_exporter`.
- **Keep** existing `helm-charts.yml` (idrac/nbu/pstore), `go-ossf-slsa3-publish.yml` (SLSA) where present — replace only `ci`/`docs`/`release`; the old inline semgrep/codeql is superseded by `go-security` (delete `nbu`'s `codeql.yml`/`static.yml` if redundant, else keep).
- Most are already SHA-pinned + modern (sibling family of pscale) → expect low iteration.

## Batch 2 — Go CLIs (3 repos)
`go-evtx`, `pdf2md`, `san-conv`.
- **`pdf2md`**: has `jekyll-gh-pages.yml` → **migrate docs to mkdocs**: create a minimal `mkdocs.yml` (material) if absent, add `make docs`, replace the jekyll workflow with `docs-publish@v1`. (Phase 4 Jekyll item, folded here.)
- **`go-evtx`**: has `pages.yml` — confirm it's docs; migrate to `docs-publish` (create `mkdocs.yml` if it was jekyll/raw pages).

## Batch 3 — Python (8 repos) + greenfield
Standard: `classifai`, `anki-maker`, `mailtag`, `lrc-automation`, `store-predict`.
- **`mailtag`**: keep `docker.yml` (container build) + `label.yml`; replace ci/docs/release.
- **`classifai`**: no publish target; ci + security + (docs if it has mkdocs).
- **PyPI publisher** `code-review-graph`: **VERIFY OWNERSHIP FIRST** — `git -C ~/Projects/code-review-graph remote -v` points to `tirth8205/...`. Confirm the push remote is Fred's fork before opening a PR; if it's not pushable as fjacquet, SKIP and flag. If owned, add `python-release@v1` (it publishes to PyPI).
- **Greenfield (no CI, additive — handle individually, NOT auto-merge):** `vault-rag-mcp` (has pyproject → add Makefile + ci/security/docs callers), `ppdm2jira` (no pyproject, source under `Ppdm2Jira/` → needs pyproject scaffolding first; treat as a mini-project, likely defer to its own task). These get **PRs left open for review** (additive CI warrants a human look), not auto-merge.

## Batch 4 — Frontend (14 repos)
`spec-search`, `ppdm-report`, `elk-sizer`, `os-sizer`, `vcf-sizer`, `icons`, `360gantt`, `converty`, `llmvram`, `presizion`, `raidy`, `vatlas`, plus `network-sizer`→GitHub **`netstack`** and `vgpu-advisor` (ci+security only).
- **`netstack`**: local dir `network-sizer`; use GitHub name `fjacquet/netstack` for all `gh` commands.
- **`vgpu-advisor`**: ci + security only; KEEP its `release.yml` (npm + Docker publish).
- **`icons`**: deploys via `npm run deploy` (gh-pages branch), not the Pages action → migrate ci + security; KEEP its deploy mechanism (don't force web-deploy). Note for review.
- **`converty`**: publish target "none"; ci + security only; KEEP `claude-code-review.yml`/`claude.yml`.
- **`spec-search`**: mixed JS+Python MCP → web-ci/web-security for the JS app; note Python MCP component has no CI.
- **`raidy`/`vatlas`**: already have codeql + SBOM → web-security supersedes; delete old `codeql.yml`/`static.yml`.

## Batch 5 — npm/node MCP libraries (ci + security only)
`Nano-Banana-MCP` (GitHub name as cloned): web-ci + web-security (codeql js-ts + osv + sbom node); KEEP its npm publish (`ci.yml`'s publish portion or separate). No web-deploy (it's a library, not a Pages app).

## Execution model
- Run **one archetype batch at a time**; within a batch, dispatch **one subagent per repo in parallel** (independent repos — safe). Cap parallelism at ~5 concurrent; queue the rest.
- Each subagent: migrate per recipe → push `ci/standardize` branch → open PR → `gh pr checks --watch` → fix **repo-side** failures (≤4 iters; never edit `fjacquet/ci`) → on green, `gh pr merge --auto --squash` (**fallback `--auto --merge` if squash is disabled on the repo**) → report. If it hits a **central bug**, STOP and report (controller fixes `fjacquet/ci`, moves `v1`, re-runs).
- **Branch-protection required-check repoint (DECIDED: auto):** if a PR is `mergeStateStatus: BLOCKED` with all visible checks green, the repo's branch protection requires an OLD check name. Repoint it to the new names so auto-merge fires: `echo '{"strict":false,"checks":[{"context":"ci / ci"},{"context":"security / security"}]}' | gh api -X PATCH repos/fjacquet/<repo>/branches/main/protection/required_status_checks --input -`. (Owner has admin. For frontend repos the contexts are still `ci / ci` + `security / security`; for repos without a security caller, use just `ci / ci`.) Check first with `gh api .../required_status_checks --jq .contexts`.
- **Older-Go cascade:** if `make tools`/`make vuln` fails (golangci-lint/goreleaser need Go 1.25; repo go.mod <1.25), bump the go directive and pin `GORELEASER_VERSION`/`GOLANGCI_VERSION` to compatible versions (cee precedent: go 1.25.11 + goreleaser v2.12.0).
- **Required-check context names per archetype (use these exact strings when repointing):**
  - Go / Python: `["ci / ci", "security / security"]` (+ `"extra"` if the repo keeps an `extra-checks.yml`).
  - Frontend: `["ci / build", "security / codeql", "security / sbom", "security / osv-scan / osv-scan"]` — web-ci's job is `build` (→ `ci / build`, NOT `ci / ci`), and web-security has three jobs (no `security / security`).
- **Conflicted-PR trap (esp. frontend):** the migration deletes old workflows (`codeql.yml`/`static.yml`/`go.yml`…) that Dependabot may have modified on `main` since branching → modify/delete conflict → PR `DIRTY` → GitHub silently runs NO `pull_request` checks. After opening the PR, verify `gh pr view --json mergeable` == `MERGEABLE`; if `CONFLICTING`, `git fetch origin && git merge origin/main`, resolve modify/delete with `git rm <old-workflow>`, push. (vsizer precedent.)
- **`gh run rerun` does NOT re-resolve `@v1`** — to pick up a moved `v1`, trigger a FRESH run (empty commit / new push), not a rerun.
- After each batch: I consolidate (merged / failed / central-bug), update the ledger, then start the next batch.
- **Greenfield repos** (`vault-rag-mcp`, `ppdm2jira`) and **ownership-uncertain** (`code-review-graph`): PRs left OPEN for your review, not auto-merged.

## Phase 4 items (folded in)
- Jekyll→mkdocs: `pdf2md`, `go-evtx` (in Batch 2). `para-files` is out of scope (not an owned app repo in the audit).
- Sonar retirement: the sonar workflows live in non-scope repos (`cfg2html` fork, `dockerfiles`, `powershell-tools`, `terraform-lab`) — out of this rollout.

## Risks & mitigations
- **Auto-merge lands a bad CI on main:** mitigated — auto-merge only fires after `*-ci`+`*-security` pass; failures stay as open PRs for triage.
- **Older-Go exporters fail `make tools`:** override tool versions in-repo (recipe documented; camt-csv precedent).
- **Greenfield repos need scaffolding judgment:** excluded from auto-merge; PRs reviewed.
- **`code-review-graph` wrong remote:** verify before push; skip if not pushable as fjacquet.
- **A central bug surfaces mid-batch:** controller fixes `fjacquet/ci`, moves `v1`; in-flight PRs re-run on the moved tag.

## Success criteria
- Every owned repo's CI is a thin caller to `fjacquet/ci@v1`; `*-ci`+`*-security` green.
- Builder drift eliminated fleet-wide; one Dependabot stream (the central repo) covers all.
- Auto-merged PRs land; open PRs (greenfield/uncertain) await review.
- `pdf2md`/`go-evtx` build docs via mkdocs.
