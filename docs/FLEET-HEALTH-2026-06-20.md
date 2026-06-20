# Fleet Health Audit + Refresh — 2026-06-20

Consolidated results of the fleet-wide health pass over every owned, in-scope repo
(all on `fjacquet/ci@v1`). Audited 5 dimensions (CLAUDE.md, ADRs, docs/mkdocs, pending
merges, release), fixed what was stale via one-PR-per-repo, cut patch releases where
warranted, standardized README badges, and remediated a fleet-wide goreleaser release
pipeline that the release tags exposed.

## Summary

- **Repos audited:** 37 (15 Go · 8 Python · 14 frontend/node). Excludes `code-review-graph`
  (not owned) and `ppdm2jira` (PowerShell, no remote).
- **Docs-refresh PRs merged:** 21 (mkdocs `extra.version` + canonical `repo_url`/`repo_name`;
  CLAUDE.md created where missing).
- **README badge PRs merged:** 26 (standard CI + release + license set).
- **Patch releases cut:** 14 Go repos (1 still finishing — see nbu).
- **CLAUDE.md created (was missing):** `vault-rag-mcp`.
- **Central fix shipped:** `go-release.yml@v1` now provisions `cosign` + `cyclonedx-gomod`;
  `v1` moved forward. Plus per-repo goreleaser fixes (cee, ppdd, nbu).
- **Flagged for human follow-up:** spec-search CLAUDE.md, pdf2md release, frontend release tags
  (see Notes).

## Execution model

Background subagents could not be granted Bash in this session, so the **controller (Opus)
ran the entire pass directly** (user-authorized), including merges and patch-release tags.
Releases were merged with `--merge` (user preference), squash as fallback.

## Go exporters & CLIs (15)

| repo | CLAUDE | docs | release (old → new) | refresh PR | badges PR | status |
|---|---|---|---|---|---|---|
| pscale_exporter | GOOD | mkdocs extra.version | v0.12.1 → **v0.12.2** | #22 | (had) | ✅ released |
| cee-exporter | GOOD | mkdocs extra.version | v4.1 → **v4.1.1** | #7 | (had) | ✅ released (+#8 goreleaser file, +#9 LICENSE) |
| go-evtx | GOOD | mkdocs extra.version | v0.5.0 → **v0.5.1** | #2 | #3 | ✅ released |
| idrac_exporter | GOOD | mkdocs extra.version | v1.0.0 → **v1.0.1** | #23 | (had) | ✅ released |
| camt-csv | GOOD | mkdocs extra.version | v2.3.2 → **v2.3.3** | #15 | #17 | ✅ released |
| nbu_exporter | GOOD | mkdocs extra.version | v4.0.0 → **v4.0.1** | #48 | (had) | re-releasing (+#49 cask token fix) |
| san-conv | GOOD | mkdocs extra.version | v1.3.0 → **v1.3.1** | #11 | #12 | ✅ released |
| nsr_exporter | GOOD | mkdocs extra.version | v0.12.1 → **v0.12.2** | #20 | (had) | ✅ released |
| ecs_exporter (gh: obs_exporter) | GOOD | mkdocs + canonical repo_name | v2.5.1 → **v2.5.2** | #9 | (had) | ✅ released |
| ppdd_exporter | GOOD | mkdocs extra.version | v0.8.1 → **v0.8.2** | #16 | #18 | ✅ released (+#17 binaries fix) |
| pmax_exporter | GOOD | mkdocs + canonical repo_name | v0.5.1 → **v0.5.2** | #9 | #10 | ✅ released |
| pstore_exporter | GOOD | mkdocs extra.version | v0.10.0 → **v0.10.1** | #19 | (had) | ✅ released |
| ppdm_exporter | GOOD | mkdocs + canonical repo_name | v2.0.1 → **v2.0.2** | #21 | #25 | ✅ released |
| pflex_exporter | GOOD | mkdocs extra.version | v0.10.2 → **v0.10.3** | #32 | (had) | ✅ released |
| pdf2md | GOOD | mkdocs extra.version | n/a (deferred) | #5 | #6 | docs only — see Notes |

## Python apps/libs (8)

| repo | CLAUDE | docs | release | refresh PR | badges PR | notes |
|---|---|---|---|---|---|---|
| anki-maker | GOOD | mkdocs extra.version | n/a | #11 | #12 | no publish mechanism |
| classifai | GOOD | mkdocs extra.version | n/a | #4 | #5 | default `maincd` |
| finwiz | GOOD | already compliant | n/a | — | #74 | extra.version already present |
| lrc-automation | GOOD | mkdocs + canonical repo_name | deferred | #11 | #12 | GH-release mechanism; tag deferred |
| mailtag | GOOD | mkdocs extra.version | deferred | #24 | (had) | GH-release mechanism; tag deferred |
| store-predict | GOOD | mkdocs extra.version | deferred | #42 | (had) | GH-release mechanism; tag deferred |
| vault-rag-mcp | **CREATED** | mkdocs repo_url/repo_name | n/a (no tag) | #2 | #3 | CLAUDE.md was MISSING |
| spec-search | FLAG | (hybrid, no mkdocs) | deferred | — | #29 | CLAUDE.md holds RTK instructions, not project docs — needs human decision |

## Frontend / node (14)

Docs GOOD (CLAUDE.md present; READMEs accurate). README badges standardized where missing.
**Release tags deferred** (varied npm/deploy mechanisms; avoided release churn after the
goreleaser remediation).

| repo | badges PR | notes |
|---|---|---|
| 360gantt | (had) | GOOD |
| converty | #10 | keeps claude review wf |
| elk-sizer | #3 | default `maincd` |
| icons | #4 | default `master`; keeps gh-pages deploy |
| llmvram | #30 | |
| network-sizer (gh: netstack) | #13 | default `maincd` |
| os-sizer | #6 | branch state fixed by user; default `main` |
| presizion | #30 | |
| raidy | #43 | |
| vatlas | #18 | |
| vcf-sizer | #72 | |
| vgpu-advisor | #7 | default `maincd`; ci+sec only |
| vsizer | #39 | |
| Nano-Banana-MCP | #2 | npm library |

## Central pipeline remediation (`fjacquet/ci`)

The patch tags exposed pre-existing goreleaser failures (the docs PRs themselves were clean).
Diagnosis and fixes:

- **PR #5 (merged), `v1` moved** — `go-release.yml` now installs `sigstore/cosign-installer`
  (+`sigstore/*` allow-listed in zizmor) and `cyclonedx-gomod`, so repo goreleaser configs
  that reference them work regardless of each repo's `make tools`. Fixed the
  `cyclonedx-gomod: not found` (idrac/pflex/ppdd/pstore) and `cosign: not found` (nbu) class.
- **cee-exporter** — had **no** `.goreleaser.yaml` (`no-main`); added one (#8) + removed a
  `LICENSE` archive glob for a file that doesn't exist (#9).
- **ppdd_exporter** — `binaries:` field invalid in its goreleaser v2.12.0 cask (#17).
- **nbu_exporter** — cask used wrong env var `TAP_GITHUB_TOKEN` (→ `HOMEBREW_TAP_GITHUB_TOKEN`)
  + `skip_upload` guard (#49); a stale partial release was deleted to clear a 422 asset conflict.

## Notes / human follow-ups

1. **spec-search** — its `CLAUDE.md` contains RTK (Rust Token Killer) instructions, not
   project documentation. Left as-is; decide whether to author a real project CLAUDE.md.
2. **pdf2md release** — uses central `go-release@v1` but needs CGO cross-compile
   (`goreleaser-cross`); a `v*` tag would fail the cross-build. Refreshed docs only; release
   deferred pending a repo-specific release.yml or a central cross-compile option.
3. **Python release tags** (lrc-automation/mailtag/store-predict) — have GH-release
   mechanisms; tags deferred (not cut this pass).
4. **Frontend release tags** — deferred for the 14 frontend repos.
5. **cee-exporter** — released as `v4.1.1` (treated the 2-component `v4.1` as `v4.1.0`).
6. **camt-csv** — local `main` carries an unpushed commit `7ce7e0f "updated with new"`
   (refresh based on `origin/main`, left untouched — review).
