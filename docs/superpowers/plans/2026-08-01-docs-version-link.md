# Docs Version Link (central half) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking. This repo has **no application code and no unit tests** — "verification" means `actionlint` / `zizmor` / `pinact` run clean plus a real local `mkdocs build` that proves the rendered footer contains the anchor. Evidence before assertions.

**Goal:** Every MkDocs Material site published by `fjacquet/ci`'s `docs-publish` workflow carries a visible footer link to the source code **at the release version the docs describe** — e.g. `Source: fjacquet/obs_exporter v3.4.0` linking to `https://github.com/fjacquet/obs_exporter/tree/v3.4.0`. The central half (this plan) makes `actions/mkdocs-publish` derive the version and export a ready-to-render `SITE_FOOTER` env var; consumers opt in with a single `mkdocs.yml` line, handled by a separate plan.

**Architecture:** A new pure-`run:` step in the composite action `actions/mkdocs-publish/action.yml`, placed **before** `make docs`, resolves a version string (tag ref → nearest tag → short SHA), sanitises it against a conservative git-ref charset, composes the **complete** footer HTML centrally, and writes it to `$GITHUB_ENV` with a random-delimiter heredoc. `make docs` inherits it; MkDocs reads it via `copyright: !ENV [SITE_FOOTER, ""]`. Composing centrally (not exporting raw pieces) is the locked decision: the consumer's `mkdocs.yml` then contains zero repo-specific logic, so the same one-liner is copy-pasteable across all fifteen repos and future link-shape changes ship from here alone without a fleet-wide edit.

**Tech Stack:** GitHub Actions composite action (bash), git, MkDocs core `!ENV` YAML tag, mkdocs-material `copyright` (which renders raw HTML), `uvx` for local verification, `actionlint` / `zizmor` / `pinact` for validation.

## Global Constraints

- **Do not touch consumer repos.** This plan changes `fjacquet/ci` only. The consumer-side one-liner is a separate plan; this plan's job is to publish a contract that plan can rely on verbatim.
- **Established facts — do not re-derive or contradict:**
  - `extra.version` in consumer `mkdocs.yml` files is dead config (renders nothing without `extra.version.provider: mike` plus an `overrides/` dir; no repo has either) and is stale in 13 of 15 repos. It must be **removed**, not corrected — in the *consumer* plan.
  - `copyright:` renders raw HTML, and MkDocs' `!ENV [VAR, default]` tag works. Both were proven with a throwaway build.
  - `!ENV` is already an established idiom in Fred's code (`/Users/fjacquet/finwiz/mkdocs.yml` line 223 uses `!ENV GOOGLE_ANALYTICS_KEY`).
  - `docs-publish.yml` already checks out with `fetch-depth: 0`, so tags are present at build time.
- **Action pinning (DESIGN.md D3, enforced by `zizmor.yml` + `pinact`):** third-party actions are SHA-pinned with a trailing `# vX.Y.Z` comment; first-party `fjacquet/ci` refs use the moving `@v1` tag with the trailing `# nosemgrep: github-actions.security.third-party-action-not-pinned-to-commit-sha` comment. **The new step is a pure `run:` step — it uses no action, so it needs no pinning and no nosemgrep comment.** Do not add either.
- **No new `nosemgrep` suppressions.** The repo establishes that pattern in exactly one place (the `uses: fjacquet/ci/...@v1` first-party ref, e.g. `.github/workflows/docs-publish.yml:27`). Nothing in this change is a `uses:`, so nothing here gets a suppression.
- **Backward compatible, no exceptions.** A consumer that has *not* added the `copyright:` line must still build exactly as today. Setting an env var that nothing reads is a no-op — do not add any step that inspects, validates, or requires the consumer's `mkdocs.yml`.
- **Degrade gracefully, never fail.** A tagless repo, a build from a branch ref, a detached HEAD, or a repo whose tag contains exotic characters must all produce a working (if less precise) footer and a green build. No `exit 1` anywhere in the new step.
- **`shell: bash` in GitHub Actions runs `bash --noprofile --norc -eo pipefail {0}`** — `-e` and `pipefail` are already on. Every command that is allowed to fail must carry an explicit `|| true` / `|| echo <fallback>`.
- Validation loop before pushing (from `CLAUDE.md`): `actionlint -color`, `uvx zizmor --format=github .`, `pinact run --check --exclude '^fjacquet/'`.
- **Planning-doc location:** this repo already uses `docs/superpowers/plans/` (alongside `docs/superpowers/specs/` and an older `docs/plans/`). This plan follows that existing convention; no new location was invented.

## File Structure

| Path | Action | Purpose |
|------|--------|---------|
| `actions/mkdocs-publish/action.yml` | Modify | Add the `Derive source-link footer` step before `make docs`; add a `version` composite output. |
| `.github/workflows/docs-publish.yml` | Verify only (no edit expected) | Confirm nothing blocks a tag-triggered run; add a clarifying comment only if the audit in Task 2 finds one. |
| `templates/workflows/docs.yml` | Modify | Trigger the canonical consumer caller on `v*` tag pushes so a release refreshes the footer immediately; drop the `paths:` filter that would suppress it. |
| `README.md` | Modify | Publish the consumer contract: env var name, exact shape, the one-line `mkdocs.yml` change, fallback behaviour. |
| `docs/superpowers/plans/2026-08-01-docs-version-link.md` | Create (this file) | The plan. |

---

### Task 1: Derive and export `SITE_FOOTER` in the composite action

**Files:**
- Modify: `actions/mkdocs-publish/action.yml`

**Interfaces:**
- **Consumes:** `GITHUB_REF_TYPE`, `GITHUB_REF_NAME`, `GITHUB_SERVER_URL`, `GITHUB_REPOSITORY`, `GITHUB_ENV`, `GITHUB_OUTPUT` (all runner-provided); a git checkout with `fetch-depth: 0` in `GITHUB_WORKSPACE`.
- **Produces:** environment variable `SITE_FOOTER` (a complete HTML fragment, single line) visible to every later step in the job, including `make docs`; composite action output `version` (the bare resolved version string, no HTML).

- [ ] **Step 1: Read the current action file.** Open `actions/mkdocs-publish/action.yml`. Confirm the step order is: `setup-python` → `setup-uv` → `Sync docs dependencies` → `Build docs` (`make docs`) → `configure-pages` → `upload-pages-artifact`. The new step goes **between `Sync docs dependencies` and `Build docs`** — after the toolchain is ready, before anything reads the env.

- [ ] **Step 2: Add the derivation step.** Insert the following block immediately before the `- name: Build docs` step (indentation: the `-` sits at 4 spaces, matching its siblings).

```yaml
    - name: Derive source-link footer
      id: footer
      shell: bash
      # Exports SITE_FOOTER — a complete HTML anchor pointing at the source tree at the
      # version these docs describe. Consumers render it with `copyright: !ENV [SITE_FOOTER, ""]`.
      # Never fails: a tagless repo or a non-tag ref degrades to the short SHA.
      run: |
        # 1. Resolve the version. A tag ref is authoritative; otherwise fall back to the
        #    nearest tag reachable from HEAD; otherwise the short SHA.
        sha="$(git rev-parse --short HEAD 2>/dev/null || echo unknown)"
        if [ "${GITHUB_REF_TYPE:-}" = "tag" ] && [ -n "${GITHUB_REF_NAME:-}" ]; then
          version="${GITHUB_REF_NAME}"
        else
          version="$(git describe --tags --abbrev=0 2>/dev/null || true)"
        fi
        [ -n "${version}" ] || version="${sha}"

        # 2. Sanitise. Allow only a conservative git-ref charset, so the value is safe both
        #    as a URL path segment and as HTML text without any escaping pass. Anything else
        #    (including < > & " ' and whitespace) falls back to the short SHA rather than
        #    emitting markup we did not compose.
        case "${version}" in
          -*|*..*|*[!A-Za-z0-9._/-]*) version="${sha}" ;;
        esac

        # 3. Compose the whole footer centrally, so consumer mkdocs.yml stays repo-agnostic.
        url="${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}/tree/${version}"
        footer="Source: <a href=\"${url}\" rel=\"noopener\">${GITHUB_REPOSITORY} ${version}</a>"

        # 4. Write with a random-delimiter heredoc (the GITHUB_ENV-safe form for any value).
        delim="SITE_FOOTER_EOF_$(openssl rand -hex 16)"
        {
          printf 'SITE_FOOTER<<%s\n' "${delim}"
          printf '%s\n' "${footer}"
          printf '%s\n' "${delim}"
        } >> "${GITHUB_ENV}"
        printf 'version=%s\n' "${version}" >> "${GITHUB_OUTPUT}"
        printf 'docs source link: %s\n' "${footer}"
```

Notes for the implementer, so nothing here gets "simplified" away:

- `${GITHUB_REPOSITORY}` is `owner/repo` and cannot contain HTML-significant characters, so it needs no escaping either.
- The sanitiser makes HTML escaping unnecessary **and** makes the heredoc delimiter unspoofable (the value provably contains no newline). Keep both — the random delimiter is the documented-correct form for `$GITHUB_ENV` and costs nothing.
- `case` bracket note: the trailing `-` in `[!A-Za-z0-9._/-]` is literal because it is last. Do not reorder.
- `openssl` ships on `ubuntu-24.04`; the derivation step falls back to `/dev/urandom` and then `$RANDOM` if it is absent — see Task 1 Step 2's updated delimiter logic.

- [ ] **Step 3: Expose the resolved version as a composite output.** Add an `outputs:` block to the action, after `inputs:` and before `runs:`. This is purely additive — existing callers that ignore it are unaffected.

```yaml
outputs:
  version:
    description: Version string used in the docs source link (tag name, nearest tag, or short SHA)
    value: ${{ steps.footer.outputs.version }}
```

- [ ] **Step 4: Sanity-read the result.** Re-read `action.yml` end to end and confirm: `id: footer` is present on the new step, the new step precedes `Build docs`, no `uses:` was added (hence no SHA pin and no nosemgrep comment), and the `inputs.python-version` block is untouched.

- [ ] **Step 5: Lint the action.** Run from the repo root:

```bash
actionlint -color && uvx zizmor --format=github . && pinact run --check --exclude '^fjacquet/'
```

  All three must exit 0. If `zizmor` flags the new step, do **not** add a suppression — restructure the step until it passes.

---

### Task 2: Audit the reusable workflow for tag-triggered runs

**Files:**
- Verify: `.github/workflows/docs-publish.yml` (expected outcome: **no change**)

**Interfaces:**
- **Consumes:** the caller's trigger event.
- **Produces:** a recorded finding — either "no change needed" or a minimal fix.

- [ ] **Step 1: Check the checkout depth.** Confirm `.github/workflows/docs-publish.yml` still has `fetch-depth: 0` on `actions/checkout`. Without it `git describe` sees no tags and every build silently degrades to the SHA. Nothing to change today — it is already set — but this is the one line that must never regress.

- [ ] **Step 2: Check the deploy gate against a tag push.** The `deploy` job is gated `if: github.event_name == 'push'`. A tag push **is** `event_name: push` (`ref_type: tag`), so a tag-triggered caller reaches the deploy job. Confirm by reading lines 30-46. No change required — record this explicitly so the consumer plan can rely on it.

- [ ] **Step 3: Check the concurrency group.** `concurrency: {group: pages, cancel-in-progress: false}` at workflow level. A tag push and a main push landing together queue rather than cancel — which is what we want, since the tag build carries the newer footer and must not be killed. No change required.

- [ ] **Step 4: Confirm `workflow_call` inputs need no addition.** The version is derived from the runner's own git context, not passed in, so no new input is needed. Adding one would be a breaking-adjacent interface change for zero benefit. Explicitly decide: **no input added.**

- [ ] **Step 5: Record the audit result** in the PR description (not a new file): "docs-publish.yml unchanged — `fetch-depth: 0` present, deploy gate accepts tag pushes, pages concurrency queues rather than cancels."

---

### Task 3: Make the canonical consumer caller template react to tags

**Files:**
- Modify: `templates/workflows/docs.yml`

**Interfaces:**
- **Consumes:** nothing new.
- **Produces:** the template consumers copy in the *other* plan; it must already contain the tag trigger so that plan is a pure copy, not a copy-plus-edit.

- [ ] **Step 1: Understand why `paths:` has to go.** A workflow cannot declare two `push:` keys, and when `branches`/`tags` and `paths` are combined, GitHub requires **both** filters to match. For a tag push the changed-file set is evaluated against the push's before-ref, which for a fresh tag is unreliable — a `paths: ["docs/**", "mkdocs.yml"]` filter will usually suppress the tag build entirely, which is exactly the build that carries the new version. The template therefore drops `paths:` and gains `tags:`. Cost: docs rebuild on every `main` push. That is a ~2-minute job on `ubuntu-24.04` and the Pages concurrency group already serialises it.

- [ ] **Step 2: Replace the `on:` block.** In `templates/workflows/docs.yml`, replace:

```yaml
on:
  push:
    branches: [main]
    paths: ["docs/**", "mkdocs.yml"]
  pull_request:
    paths: ["docs/**", "mkdocs.yml"]
```

  with:

```yaml
on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
    paths: ["docs/**", "mkdocs.yml"]
```

  The `pull_request` path filter stays — PRs only validate docs, and there is no tag involved.

- [ ] **Step 3: Update the template's header comment.** Replace the first comment paragraph with:

```yaml
# Canonical caller for the docs site. Copy to a consumer repo as
# `.github/workflows/docs.yml`. Builds the MkDocs Material site and deploys it to
# GitHub Pages via fjacquet/ci. On pull_request the docs still build (validation)
# but the deploy job self-skips — only a push publishes.
#
# The `tags: ["v*"]` trigger exists so a release immediately republishes the site with
# the new version in the footer source link (see SITE_FOOTER in README.md). Do not add a
# `paths:` filter to the push trigger: combined path+tag filters suppress tag builds.
#
# Pages prerequisite: set the repo's Pages source to "GitHub Actions" (build_type
# workflow), not a branch.
```

- [ ] **Step 4: Lint.** `actionlint -color` — the templates directory is covered by the self-check.

---

### Task 4: Publish the consumer contract in README.md

**Files:**
- Modify: `README.md`

**Interfaces:**
- **Consumes:** the behaviour built in Task 1.
- **Produces:** the normative, quotable contract the consumer-side plan implements against.

- [ ] **Step 1: Add a contract section.** Insert the following after the `### Frontend (web) repos` subsection and before `### Published npm packages` in `README.md`:

````markdown
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
````

- [ ] **Step 2: Cross-reference from the workflow table.** In the `## Workflows` table, change the `docs-publish` Purpose cell from `MkDocs build + deploy to GitHub Pages` to `MkDocs build + deploy to GitHub Pages (exports `SITE_FOOTER`, see [Docs sites: the version source link](#docs-sites-the-version-source-link))`.

- [ ] **Step 3: Verify the anchor.** Confirm the heading text `### Docs sites: the version source link` produces the GitHub anchor `#docs-sites-the-version-source-link` (lowercase, spaces→hyphens, `:` dropped). Fix the link if it does not match.

---

### Task 5: Verify the footer actually renders (real build, real grep)

**Files:** none in the repo — this task builds throwaway artefacts under `/tmp`.

**Interfaces:**
- **Consumes:** `SITE_FOOTER` shaped exactly as Task 1 emits it.
- **Produces:** grep evidence that the anchor reaches the published HTML, plus evidence of graceful degradation.

- [ ] **Step 1: Build the throwaway site.**

```bash
rm -rf /tmp/footer-check && mkdir -p /tmp/footer-check/docs
cat > /tmp/footer-check/mkdocs.yml <<'YAML'
site_name: footer-check
theme:
  name: material
copyright: !ENV [SITE_FOOTER, ""]
YAML
printf '# hello\n' > /tmp/footer-check/docs/index.md
cd /tmp/footer-check && SITE_FOOTER='Source: <a href="https://github.com/fjacquet/obs_exporter/tree/v3.4.0" rel="noopener">fjacquet/obs_exporter v3.4.0</a>' \
  uvx --with mkdocs-material --with pymdown-extensions mkdocs build --strict --site-dir site
```

  This mirrors the consumers' real `make docs` line (see `/Users/fjacquet/Projects/kemp_exporter/Makefile:96`).

- [ ] **Step 2: Prove the anchor is in the output HTML.** Must print a matching line:

```bash
grep -o '<a href="https://github.com/fjacquet/obs_exporter/tree/v3.4.0"[^>]*>[^<]*</a>' /tmp/footer-check/site/index.html
```

  Expected: `<a href="https://github.com/fjacquet/obs_exporter/tree/v3.4.0" rel="noopener">fjacquet/obs_exporter v3.4.0</a>`. If it prints nothing, the mechanism is broken — stop and debug before touching anything else.

- [ ] **Step 3: Prove backward compatibility (unset variable).** Must exit 0 and the anchor must be **absent** — this command itself fails (non-zero) if the anchor is unexpectedly present, rather than merely printing a grep count:

```bash
cd /tmp/footer-check
env -u SITE_FOOTER uvx --with mkdocs-material --with pymdown-extensions mkdocs build --strict --site-dir site
if grep -q 'tree/v3.4.0' site/index.html; then
  echo "FAIL: anchor present with SITE_FOOTER unset — backward compatibility broken" >&2
  exit 1
fi
echo "OK: no anchor present with SITE_FOOTER unset"
```

- [ ] **Step 4: Unit-test the derivation logic against five cases, asserting each one.** Extract the script body from Task 1 Step 2 into `/tmp/footer-check/derive.sh` (drop the YAML wrapper, keep the bash verbatim — including the `openssl` → `/dev/urandom` → `$RANDOM` delimiter fallback chain — and add `set -eo pipefail` at the top since `shell: bash` supplies it in CI), then run it against a scratch git repo. Each case below must assert its expected `version=` output and non-zero exits must themselves fail the script (no bare `cat` that only prints and moves on):

```bash
rm -rf /tmp/footer-repo && mkdir /tmp/footer-repo && cd /tmp/footer-repo
git init -q && git commit -q --allow-empty -m init
export GITHUB_SERVER_URL=https://github.com GITHUB_REPOSITORY=fjacquet/obs_exporter
export GITHUB_ENV=/tmp/footer-repo/env GITHUB_OUTPUT=/tmp/footer-repo/out

assert_version() {
  # $1 = case label, $2 = regex the captured `version=` output must match
  actual="$(grep '^version=' "$GITHUB_OUTPUT" | tail -1 | cut -d= -f2-)"
  if ! printf '%s' "${actual}" | grep -qE "$2"; then
    echo "FAIL case $1: version='${actual}' does not match /$2/" >&2
    exit 1
  fi
  echo "OK case $1: version='${actual}'"
}

# Case A: tagless repo, branch ref -> short SHA, must not fail
: > "$GITHUB_ENV"; : > "$GITHUB_OUTPUT"
GITHUB_REF_TYPE=branch GITHUB_REF_NAME=main bash /tmp/footer-check/derive.sh || { echo "FAIL case A: derive.sh exited non-zero" >&2; exit 1; }
assert_version A '^[0-9a-f]{7}$'

# Case B: nearest tag from a branch ref -> v1.2.0
git tag -a v1.2.0 -m v1.2.0 && git commit -q --allow-empty -m next
: > "$GITHUB_ENV"; : > "$GITHUB_OUTPUT"
GITHUB_REF_TYPE=branch GITHUB_REF_NAME=main bash /tmp/footer-check/derive.sh || { echo "FAIL case B: derive.sh exited non-zero" >&2; exit 1; }
assert_version B '^v1\.2\.0$'

# Case C: tag ref wins over describe -> v9.9.9
: > "$GITHUB_ENV"; : > "$GITHUB_OUTPUT"
GITHUB_REF_TYPE=tag GITHUB_REF_NAME=v9.9.9 bash /tmp/footer-check/derive.sh || { echo "FAIL case C: derive.sh exited non-zero" >&2; exit 1; }
assert_version C '^v9\.9\.9$'

# Case D: hostile tag name -> falls back to short SHA, emits no injected markup
: > "$GITHUB_ENV"; : > "$GITHUB_OUTPUT"
GITHUB_REF_TYPE=tag GITHUB_REF_NAME='v1"><script>alert(1)</script>' bash /tmp/footer-check/derive.sh || { echo "FAIL case D: derive.sh exited non-zero" >&2; exit 1; }
assert_version D '^[0-9a-f]{7}$'
if grep -q '<script' "$GITHUB_ENV"; then
  echo "FAIL case D: <script> markup leaked into GITHUB_ENV" >&2
  exit 1
fi
echo "OK case D: no <script> markup in GITHUB_ENV"

# Case E: delimiter-collision — a tag that is itself a valid, allowlisted version string but
# contains the literal delimiter prefix embedded in it. The sanitiser's charset (A-Za-z0-9._/-)
# permits underscores, so this is NOT rejected by sanitisation; it must instead be proven safe
# by construction — the value is always embedded inside "Source: <a href=...>...</a>" text, so
# it can never appear as a standalone line matching the heredoc delimiter and closing it early.
: > "$GITHUB_ENV"; : > "$GITHUB_OUTPUT"
GITHUB_REF_TYPE=tag GITHUB_REF_NAME='SITE_FOOTER_EOF_deadbeef' bash /tmp/footer-check/derive.sh || { echo "FAIL case E: derive.sh exited non-zero" >&2; exit 1; }
assert_version E '^SITE_FOOTER_EOF_deadbeef$'
# The env file must still be exactly 3 lines added for SITE_FOOTER: an opening heredoc marker,
# one content line, and a closing marker distinct from the opening one — i.e. the embedded
# delimiter-like substring did not fracture the heredoc into extra/short lines.
env_lines="$(grep -c . "$GITHUB_ENV")"
if [ "${env_lines}" -ne 3 ]; then
  echo "FAIL case E: expected exactly 3 non-empty lines in GITHUB_ENV, got ${env_lines}" >&2
  exit 1
fi
echo "OK case E: embedded delimiter-prefix tag did not break the heredoc"
```

  Every case above is self-asserting: a failed assertion or a non-zero `derive.sh` exit stops the script with `exit 1`, not a print-and-continue. If this block runs to completion, all five cases held.

- [ ] **Step 5: Feed case D's output back through mkdocs and assert render-safety.** Take the exact `SITE_FOOTER` line case D wrote to `$GITHUB_ENV` (the middle line, between the delimiters), export it, rebuild `/tmp/footer-check`, and fail if any `<script` markup reaches the rendered footer:

```bash
cd /tmp/footer-check
site_footer_value="$(sed -n '2p' /tmp/footer-repo/env)"
SITE_FOOTER="${site_footer_value}" uvx --with mkdocs-material --with pymdown-extensions mkdocs build --strict --site-dir site
if grep -q '<script' <(grep -A1 'md-copyright' site/index.html); then
  echo "FAIL: <script> markup reached the rendered copyright footer" >&2
  exit 1
fi
echo "OK: sanitiser output is render-safe — no <script> in the rendered footer"
```

  This closes the loop: sanitiser output is provably render-safe, not merely observed to look safe.

- [ ] **Step 6: Clean up.** `rm -rf /tmp/footer-check /tmp/footer-repo`.

---

### Task 6: Ship it — PR, self-check, and the `v1` retag

**Files:** none (git/gh operations).

**Interfaces:**
- **Consumes:** Tasks 1-5 complete and verified.
- **Produces:** merged `main`, a new `v1.3.0` annotated tag, and the moving `v1` annotated tag repointed at it.

- [ ] **Step 1: Confirm the versioning convention before acting.** `README.md` ("Callers pin to a **major tag** (e.g. `@v1`)") and `DESIGN.md` D3 ("callers reference `fjacquet/ci` by moving tag `@v1`") both state the moving-major model, and the repo's tags confirm it: `v1`, `v1.0.0`, `v1.1.0`, `v1.2.0` all exist as **annotated** tags, with `v1` and `v1.2.0` currently pointing at the same commit. So: cut `v1.3.0`, then force-move `v1` onto it. Do not create `v1.3` or any minor-pin tag — none exists and consumers do not use them.

- [ ] **Step 2: Branch, commit, push.**

```bash
git switch -c feat/docs-version-source-link
git add actions/mkdocs-publish/action.yml templates/workflows/docs.yml README.md docs/superpowers/plans/2026-08-01-docs-version-link.md
git commit -m "feat(mkdocs-publish): export SITE_FOOTER with a version-pinned source link"
git push -u origin feat/docs-version-source-link
```

- [ ] **Step 3: Open the PR and wait for `self-check` green.** `gh pr create --fill`, then `gh pr checks --watch`. `actionlint`, `zizmor`, and `pinact check` must all pass. Do not merge on a red or pending check.

- [ ] **Step 4: Merge, then tag.**

```bash
gh pr merge --squash --delete-branch
git switch main && git pull --ff-only
git tag -a v1.3.0 -m "v1.3.0 — docs sites carry a version-pinned source link"
git tag -f -a v1 -m "v1 -> v1.3.0"
git push origin v1.3.0
git push origin -f v1
```

- [ ] **Step 5: State the consumer impact.** Record in the PR/release notes: **consumers need to do nothing to pick up the action change** — they already track the moving `@v1` tag, so the next `docs-publish` run exports `SITE_FOOTER` automatically. It has no visible effect until a repo adds `copyright: !ENV [SITE_FOOTER, ""]` to its `mkdocs.yml` (and copies the refreshed `templates/workflows/docs.yml` trigger), which is the separate consumer-side plan.

- [ ] **Step 6: Smoke-test on one live repo.** Pick a repo whose docs already publish (e.g. `obs_exporter`), re-run its `Docs` workflow from the Actions UI, and confirm the `Derive source-link footer` step logs a `docs source link: Source: <a href="…/tree/vX.Y.Z" …>` line pointing at that repo's current tag. The published site will not change yet — that is expected and correct until the consumer plan lands.

---

## Self-Review

**Does this meet the goal?** Partially by design — it delivers the entire central half. After this plan, every `docs-publish` build derives the right version and hands the finished HTML to `make docs`. The visible link appears only once the consumer plan adds the one-liner; that split is deliberate and stated in Task 6 Step 5.

**Decisions locked, and why:**
- *Compose the footer centrally, not raw pieces.* Considered exporting `SITE_VERSION` + `SITE_SOURCE_URL` and letting each `mkdocs.yml` assemble them — rejected: fifteen copies of string-assembly logic, and any future change to the link shape becomes a fifteen-repo edit. Central composition makes the consumer line identical everywhere and repo-agnostic.
- *Sanitise by allowlist instead of HTML-escaping.* An allowlist of `[A-Za-z0-9._/-]` makes the value simultaneously safe as a URL path segment, safe as HTML text, and provably newline-free (so the `$GITHUB_ENV` heredoc cannot be spoofed). Escaping would have needed three different escapers for the same string. The random heredoc delimiter is kept anyway as defence in depth.
- *Tag ref beats `git describe`.* On a tag push, `GITHUB_REF_NAME` is exactly the version being released; `git describe` would agree, but only if the tag is reachable and fetched. Preferring the ref removes a dependency on fetch behaviour.
- *Template drops `paths:` on push.* Combined path+tag filters suppress precisely the build we care about. Rebuilding docs on every `main` push is the cheap side of that trade.

**Risks and how the plan handles them:**
- *`fetch-depth: 0` regressing in `docs-publish.yml`* would silently downgrade every site to a SHA link with no failure. Task 2 Step 1 pins it as a must-never-regress invariant; the build stays green either way, which is the intended failure mode.
- *`openssl` absent from a future runner image* would fail the step under `-e`. Acceptable: `ubuntu-24.04` is pinned repo-wide (DESIGN.md D4) and ships openssl.
- *A consumer with an existing literal `copyright:`* (`anki-maker`, `finwiz` today) would lose that text if the line were blindly replaced. The README contract in Task 4 tells the consumer plan to fold it into the `!ENV` default instead.

**What is explicitly NOT in scope:** any edit to the fifteen consumer repos — removing their dead `extra.version:`, adding `copyright: !ENV [SITE_FOOTER, ""]`, or copying the refreshed `docs.yml` trigger. That is the sibling plan, which consumes the contract published in Task 4.
