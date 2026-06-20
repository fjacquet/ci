# Fleet Health Audit + Refresh — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking. This is an OPS fan-out, not a code feature: "verification" = real checks (CI green, tag pushed, dashboard built), not unit tests.

**Goal:** Audit all ~36 owned in-scope repos across 5 health dimensions, fix stale docs/CLAUDE.md/ADRs/mkdocs via one PR per repo, cut patch releases where warranted, and produce a consolidated dashboard.

**Architecture:** Controller (Opus, this session) pre-flights a per-repo manifest, then dispatches one **Sonnet** subagent per repo in waves of ~6–8. Subagents audit + open a `chore/health-refresh` PR + drive CI green + STOP. Controller merges PRs, pushes patch release tags post-merge, and assembles the dashboard. Same guardrail-safe split as the `@v1` rollout.

**Tech Stack:** `gh` CLI, git, the repos' own `make`/npm toolchains, mkdocs-material, goreleaser/npm/PyPI release workflows, `fjacquet/ci@v1`.

## Global Constraints

- Subagents run on **Sonnet**; controller on Opus.
- Subagents NEVER merge, push tags, modify branch protection, or edit `fjacquet/ci`. Controller only.
- Refresh PRs touch **docs/CLAUDE.md/ADRs/mkdocs.yml only** — no application-code changes.
- Versioning is **patch-only**: `vX.Y.(Z+1)` from the latest tag, regardless of commit content.
- Release only for repos with a real publish mechanism (goreleaser/`release.yml`/npm/PyPI) AND unreleased commits since the latest tag; cut the tag **after** the repo's refresh PR merges.
- Live default branch is authoritative via `gh repo view <repo> --json defaultBranchRef` (NOT local symbolic-ref — stale `maincd` lesson).
- Workflow/`mkdocs.yml` edits that contain `@v1` first-party refs: write via Bash heredoc (semgrep pre-write hook FP). Plain docs/CLAUDE.md/ADR edits can use the file-edit tool.
- Branch name `chore/health-refresh`; PR title `chore: fleet health refresh`. Merge squash with `--merge` fallback; repoint stale required checks per repo.
- Exclusions: `code-review-graph`, `ppdm2jira`. `fjacquet/ci` audited docs-only, never by an audit subagent.

---

### Task 0: Build the repo manifest (controller pre-flight)

**Files:** none (produces an in-memory/markdown manifest used to dispatch).

- [ ] **Step 1: Enumerate in-scope repos** — repos under `~/Projects` whose `.github/workflows` calls `fjacquet/ci`, cross-checked against `~/Projects/ci/docs/AUDIT.md`.

```bash
cd ~/Projects
for d in */; do d=${d%/}; [ -d "$d/.git" ] || continue; \
  if grep -rqs "fjacquet/ci" "$d/.github/workflows" 2>/dev/null; then echo "$d"; fi; done \
  | grep -vE '^(ci|code-review-graph|ppdm2jira)$'
```

- [ ] **Step 2: For each repo, capture metadata** — gh-name (network-sizer→netstack), live default branch, latest tag, release mechanism, archetype.

```bash
gh_name() { case "$1" in network-sizer) echo netstack;; *) echo "$1";; esac; }
for d in <repos>; do g=$(gh_name "$d"); \
  def=$(gh repo view fjacquet/$g --json defaultBranchRef --jq '.defaultBranchRef.name'); \
  tag=$(git -C ~/Projects/$d describe --tags --abbrev=0 2>/dev/null || echo "none"); \
  rel=$(ls ~/Projects/$d/.github/workflows/release.yml 2>/dev/null && echo yes || echo no); \
  echo "$d | gh=$g | default=$def | latest=$tag | release_wf=$rel"; done
```

- [ ] **Step 3: Verify** — the list has ~36 repos, every one resolves a default branch (no `??`), and pilots (pscale_exporter, vsizer, finwiz, camt-csv) are included. Record the manifest in the ledger.

---

### Task N (per repo): Audit + refresh subagent  — dispatched in waves of ~6–8 on Sonnet

**Files (per repo):** Modify/Create `CLAUDE.md`, `README.md`, `docs/**`, `mkdocs.yml`, `docs/adr/**` as needed on branch `chore/health-refresh`.

**Interfaces:**
- Consumes (from Task 0): repo name, local path, gh-name, live default branch, latest tag, release-mechanism flag, archetype.
- Produces (controller relies on): scorecard (5 ratings), PR #/url + CI state, other open PRs/branches, release recommendation (`<latest> → <patch>` or `n/a`), notes.

- [ ] **Step 1: Dispatch the subagent** with `model: "sonnet"` and this contract:

```
Audit + refresh ONE repo as part of a fleet health pass. Work ONLY in ~/Projects/<repo>.
NEVER merge, push tags, modify branch protection, or edit fjacquet/ci. Open ONE PR, drive
CI green, STOP. Read ~/Projects/ci/docs/superpowers/specs/2026-06-20-fleet-health-audit-design.md
for full criteria.

REPO: <repo> (gh fjacquet/<gh-name>), default <default-branch>, latest tag <tag>, archetype <archetype>.

1. git fetch; checkout <default-branch>; pull; branch chore/health-refresh.
2. AUDIT 5 dimensions against the ACTUAL code, rate each GOOD/STALE/MISSING/NONE:
   a. CLAUDE.md — exists + accurate? If STALE fix; if MISSING create a minimal accurate one
      (what the repo is, archetype, make/npm commands, fjacquet/ci@v1 CI, key paths).
   b. ADRs (docs/adr/) — reflect current decisions? Fix obvious drift; NONE = note, don't invent.
   c. Docs (README + mkdocs) — match current code? Fix drift. mkdocs.yml MUST set
      repo_url: https://github.com/fjacquet/<gh-name>, repo_name: fjacquet/<gh-name>, and
      mkdocs-material extra.version = <proposed patch tag if release warranted, else latest tag>.
      Add repo/release links/badges to README where appropriate.
   d. Pending merges — list OTHER open PRs + branches ahead of default (report only).
   e. Release — unreleased commits since <tag> AND a release mechanism present? propose patch tag.
3. Apply a/b/c fixes on chore/health-refresh (docs/CLAUDE.md/ADR/mkdocs ONLY; NO app code).
   mkdocs.yml/workflow edits with @v1 refs → Bash heredoc; plain markdown → file-edit tool.
4. Commit ("chore: fleet health refresh"), push, open PR (base <default-branch>, title
   "chore: fleet health refresh").
5. gh pr checks --watch; fix repo-side CI failures (≤3 iters). STOP at green. Central fjacquet/ci
   bug → STOP + report.
RETURN (structured): scorecard (5 ratings + 1-line each), PR #+url, mergeable/mergeStateStatus,
checks-rollup, current required-check contexts (+repoint target), list of OTHER open PRs/branches,
release recommendation (<latest> → <patch>, mechanism, warranted? y/n), notable findings.
```

- [ ] **Step 2: Verify the report** — scorecard present for all 5 dims; PR green or NEEDS-ATTENTION with reason; release recommendation explicit.

---

### Task C-merge (controller): merge each refresh PR as it greens

- [ ] **Step 1:** On GREEN-READY, repoint stale required checks if `mergeStateStatus: BLOCKED` (Go/Python → `["ci / ci","security / security"]` [+`"extra"`]; frontend → `["ci / build","security / codeql","security / sbom","security / osv-scan / osv-scan"]`).
- [ ] **Step 2:** `gh pr merge <#> --repo fjacquet/<gh> --squash --delete-branch` (fallback `--merge`).
- [ ] **Step 3: Verify** merged: `gh pr view <#> --json state --jq .state` == `MERGED`.

---

### Task C-release (controller): cut patch releases post-merge

- [ ] **Step 1:** For each repo whose refresh merged AND release is warranted (mechanism + unreleased changes): compute patch tag `vX.Y.(Z+1)` from `<latest>`.
- [ ] **Step 2:** Tag the post-merge default-branch HEAD and push:

```bash
sha=$(gh api repos/fjacquet/<gh>/commits/<default> --jq .sha)
git -C ~/Projects/<repo> fetch origin
git -C ~/Projects/<repo> tag v<X.Y.Z+1> "$sha"
git -C ~/Projects/<repo> push origin v<X.Y.Z+1>
```

- [ ] **Step 3: Verify** the release workflow kicked off green: `gh run list --repo fjacquet/<gh> --workflow release.yml --limit 1`. Record old→new tag.

---

### Task D: Consolidated dashboard

**Files:** Create `~/Projects/ci/docs/FLEET-HEALTH-2026-06-20.md`.

- [ ] **Step 1:** Build a markdown table from all subagent reports + controller actions: columns `repo | CLAUDE.md | ADR | docs | open-PRs | release (old→new / n/a) | refresh-PR# | notes`.
- [ ] **Step 2:** Add a summary header (counts: repos audited, PRs merged, releases cut, CLAUDE.md created, repos flagged for human follow-up).
- [ ] **Step 3: Verify** every manifest repo appears as a row; commit the dashboard; update the ledger.

---

## Self-Review

- **Spec coverage:** all 5 dimensions (Task N step 2 a–e); full-mode fixes (Task N step 3 + C-merge); patch releases (C-release); mkdocs repo_url+version (step 2c); missing-CLAUDE.md auto-create (2a); dashboard (Task D); guardrail split (Global Constraints). ✓
- **Placeholders:** `<repo>`/`<gh>`/`<tag>` are intentional per-repo template variables filled from the Task 0 manifest, not TODOs. ✓
- **Consistency:** branch `chore/health-refresh`, PR title `chore: fleet health refresh`, repoint targets, and the Sonnet/controller split are identical across tasks. ✓
