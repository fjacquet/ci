# CI/CD Audit Inventory — Phase 0

Generated: 2026-06-19  
Scope: all owned repos under `/Users/fjacquet/Projects/` listed in the task brief.  
Branch: `ci-foundation`

## Classification Key

| Archetype | Criteria |
|-----------|----------|
| `go-exporter` | Go project exposing Prometheus metrics; releases a Docker image / Helm chart |
| `go-cli` | Go project producing a CLI binary (no metrics endpoint) |
| `python` | Primary language Python; publishes to PyPI or runs as a service/tool |
| `frontend` | Primary language TypeScript/JavaScript; deploys to GitHub Pages or npm |
| `excluded-fork` | `isFork=true` — upstream-owned; not in scope for standardisation |

## Inventory Table

| repo | archetype | fork? | primary lang | current workflows | publish target | SAST | SBOM? | notes |
|------|-----------|-------|--------------|-------------------|---------------|------|-------|-------|
| cee-exporter | go-exporter | no | Go | ci.yml, docs.yml, release.yml | pages-docs (mkdocs gh-deploy) | none | no | |
| ecs_exporter | excluded-fork | yes | Go | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | fork — upstream unknown (parent field null in API) |
| idrac_exporter | excluded-fork | yes | Go | ci.yml, docs.yml, helm-charts.yml, release.yml | pages-docs | none | yes (SLSA) | fork — upstream unknown (parent field null in API); archetype=excluded-fork per brief rule |
| nbu_exporter | go-exporter | no | Go | ci.yml, codeql.yml, helm-charts.yml, release.yml, static.yml | pages-docs (mkdocs gh-deploy) | codeql | yes (SLSA) | |
| nsr_exporter | go-exporter | no | Go | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | |
| pflex_exporter | go-exporter | no | Go | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | |
| pmax_exporter | go-exporter | no | Go | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | |
| ppdd_exporter | go-exporter | no | Go | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | |
| ppdm_exporter | go-exporter | no | Go | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | |
| pscale_exporter | go-exporter | no | Go | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | |
| pstore_exporter | go-exporter | no | Go | ci.yml, docs.yml, helm-charts.yml, release.yml | pages-docs | none | yes (SLSA) | |
| camt-csv | go-cli | no | Go | docs.yml, go-ossf-slsa3-publish.yml, go.yml, goreleaser.yml | pages-docs | none | yes (SLSA3) | |
| go-evtx | go-cli | no | Go | ci.yml, pages.yml, release.yml | pages-docs | none | no | |
| pdf2md | go-cli | no | Go | go.yml, jekyll-gh-pages.yml, release.yml | pages-docs | none | no | |
| san-conv | go-cli | no | Go | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | |
| spec-search | frontend | no | JavaScript | ci.yml, release.yml | pages-app | none | yes (CycloneDX) | mixed stack: JS frontend + Python MCP server; SBOM covers both |
| finwiz | python | no | Python | docs.yml, osv-scanner.yml, quality.yml, supply-chain.yml | pages-docs | osv-scanner, supply-chain | yes (supply-chain) | |
| classifai | python | no | Python | ci.yml, dependency-review.yml, greetings.yml, summary.yml | none | dep-review | no | |
| anki-maker | python | no | Python | ci.yml, configs, docs.yml | pages-docs | none | no | |
| mailtag | python | no | Python | ci.yml, docker.yml, docs.yml, greetings.yml, label.yml, release.yml | pages-docs | none | no | |
| lrc-automation | python | no | Python | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | |
| code-review-graph | python | no (local fork of tirth8205/code-review-graph) | Python | ci.yml, publish.yml | pypi | none | no | remote origin points to tirth8205/code-review-graph.git; gh API returns fork=? (repo name mismatch); local fork, publish.yml deploys to PyPI |
| Nano-Banana-MCP | excluded-fork | yes | JavaScript | ci.yml | npm | none | no | fork (isFork=true); package name @fjacquet/nano-banana-mcp; publishes to npm — needs npm-release decision (Phase 3) |
| store-predict | python | no | Python | ci.yml, docs.yml, release.yml | pages-docs | none | yes (SLSA) | no go.mod found; pyproject.toml present → python archetype |
| ppdm-report | frontend | no | TypeScript | ci.yml, deploy.yml | pages-app | none | no | deploy.yml builds Vite app and deploys to GitHub Pages (VITE_BASE=/ppdm-report/); pages-app not pages-docs |
| ppdm2jira | python | no | Python | (none) | none | none | no | no CI workflows; no pyproject.toml yet; contains Python source under Ppdm2Jira/; greenfield (no CI yet) |
| vault-rag-mcp | python | no | Python | (none) | none | none | no | pyproject.toml present; greenfield (no CI yet) |
| elk-sizer | frontend | no | TypeScript | deploy.yml, release.yml | pages-app | none | no | |
| network-sizer | frontend | no | TypeScript | ci.yml, deploy.yml, release.yml | pages-app | none | yes (SLSA) | local dir is fjacquet/netstack on GitHub (name mismatch); gh API fork=false |
| os-sizer | frontend | no | TypeScript | ci.yml, deploy.yml | pages-app | none | no | |
| vcf-sizer | frontend | no | TypeScript | ci.yml, deploy.yml | pages-app | none | no | |
| vsizer | frontend | no | TypeScript | codeql.yml, container.yml, static.yml | pages-app | codeql | yes (SLSA) | |
| icons | frontend | no | TypeScript | ci.yml, deploy.yml | pages-app | none | no | deploy via `npm run deploy` (gh-pages branch push) |
| 360gantt | frontend | no | TypeScript | ci.yml, release.yml, static.yml | pages-app | none | no | |
| converty | frontend | no | TypeScript | claude-code-review.yml, claude.yml, release.yml, security.yml, static.yml | none | security | no | release.yml builds but no Pages or npm publish step detected |
| llmvram | frontend | no | TypeScript | ci.yml, static.yml | pages-app | none | no | |
| presizion | frontend | no | TypeScript | deploy.yml | pages-app | none | no | |
| raidy | frontend | no | TypeScript | codeql.yml, static.yml | pages-app | codeql | yes (SBOM via static.yml) | |
| vatlas | frontend | no | TypeScript | codeql.yml, static.yml | pages-app | codeql | yes (SBOM via static.yml) | |
| vgpu-advisor | frontend | no | TypeScript | release.yml, static.yml | pages-app + npm | none | no | publishes to npm (GitHub Packages) AND Docker (ghcr.io); needs npm-release decision (Phase 3) |

## Summary

| Archetype | Count | Repos |
|-----------|-------|-------|
| go-exporter | 10 | cee-exporter, nbu_exporter, nsr_exporter, pflex_exporter, pmax_exporter, ppdd_exporter, ppdm_exporter, pscale_exporter, pstore_exporter + (ecs/idrac excluded) |
| go-cli | 4 | camt-csv, go-evtx, pdf2md, san-conv |
| python | 9 | finwiz, classifai, anki-maker, mailtag, lrc-automation, code-review-graph, store-predict, ppdm2jira, vault-rag-mcp |
| frontend | 16 | spec-search, ppdm-report, elk-sizer, network-sizer, os-sizer, vcf-sizer, vsizer, icons, 360gantt, converty, llmvram, presizion, raidy, vatlas, vgpu-advisor + Nano-Banana-MCP(excl) |
| excluded-fork | 3 | ecs_exporter, idrac_exporter, Nano-Banana-MCP |
| **Total** | **42** | |

## Ambiguous Repo Resolutions

| repo | question | resolution | evidence |
|------|----------|------------|----------|
| idrac_exporter | fork or owned? | `excluded-fork` | `gh repo view fjacquet/idrac_exporter --json isFork` → `true` |
| ecs_exporter | fork or owned? | `excluded-fork` | `gh repo view fjacquet/ecs_exporter --json isFork` → `true` |
| store-predict | go-cli or python? | `python` | No `go.mod` found; `pyproject.toml` present at repo root |
| ppdm-report | pages-app or pages-docs? | `frontend` / `pages-app` | `deploy.yml` builds a Vite app (`npm run build`, `VITE_BASE=/ppdm-report/`) and deploys compiled `dist/` to Pages — this is an app, not documentation |
| ppdm2jira | python, greenfield? | `python` (no CI yet) | Python source in `Ppdm2Jira/` subfolder; no `pyproject.toml` yet; zero workflow files — greenfield (no CI yet) |
| vault-rag-mcp | python, greenfield? | `python` (no CI yet) | `pyproject.toml` present; zero workflow files — greenfield (no CI yet) |
| vgpu-advisor | needs npm decision? | `frontend` | `release.yml` publishes to npm (GitHub Packages) AND Docker (ghcr.io); needs npm-release decision (Phase 3) |
| code-review-graph | owned or fork? | `python` (local fork) | Local remote points to `tirth8205/code-review-graph.git`; `publish.yml` deploys to PyPI under fjacquet account; treated as owned for CI purposes |
| network-sizer | name mismatch? | `frontend` | Local dir `network-sizer` → GitHub remote `fjacquet/netstack`; `gh repo view fjacquet/netstack` confirms `isFork=false` |
| Nano-Banana-MCP | fork with npm publish? | `excluded-fork` | `isFork=true`; package.json name `@fjacquet/nano-banana-mcp`; CI only (no publish workflow found); excluded-fork archetype |
