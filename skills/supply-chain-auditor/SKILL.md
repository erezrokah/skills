---
name: supply-chain-auditor
description: Audit a repository for supply chain security risks. Use when the user asks to audit supply chain security, check dependency pinning, review GitHub Actions security, or harden CI/CD. Activate for partial matches like "are my deps pinned?" or "is my CI safe?".
---

# Supply Chain Security Auditor

## Phase 1: Repo Detection

Detect repository characteristics using Glob and Grep. Record which categories apply — skip inapplicable checks silently.

| Signal file(s) | Category flag |
| --- | --- |
| `package.json` | `js` |
| `pnpm-lock.yaml` OR `pnpm-workspace.yaml` | `js-pnpm` |
| `yarn.lock` | `js-yarn` |
| `package-lock.json` | `js-npm` |
| `pyproject.toml` OR `requirements.txt` OR `setup.py` | `python` |
| `uv.lock` OR `[tool.uv]` in `pyproject.toml` | `python-uv` |
| `Cargo.toml` | `rust` |
| `go.mod` | `go` |
| `.github/workflows/*.yml` or `.github/workflows/*.yaml` | `github-actions` |
| `.github/dependabot.yml` | `dependabot` |
| `renovate.json` OR `renovate.json5` OR `.renovaterc` OR `.renovaterc.json` OR `renovate` key in `package.json` | `renovate` |
| `.github/PULL_REQUEST_TEMPLATE.md` OR `.github/PULL_REQUEST_TEMPLATE/` | `has-pr-template` |
| `Dockerfile` OR `docker-compose.yml` OR `docker-compose.yaml` | `docker` |

**Fallback:** If `dependabot` not detected, run `gh pr list --author 'app/dependabot' --state all --limit 1` — if results, set `dependabot-pr-only`. Same for `renovate` with `--author 'app/renovate'` → `renovate-pr-only`.

## Phase 2: Audit Checks

Severity levels: **CRITICAL** (actively exploitable) · **HIGH** (one step from exploitation) · **MEDIUM** (defense-in-depth gap) · **LOW** (best practice missing) · **PASS** (no issue)

---

### Category 1: CI Issues — code fixes applied in PR

Findings in this category **MUST be fixed with actual code changes** committed in the PR branch.

#### 1.1: GitHub Actions SHA Pinning

**Applies to:** `github-actions`

Search `.github/workflows/*.yml`, `.github/workflows/*.yaml`, and `.github/actions/*/action.yml` for `uses:` directives.
- **PASS:** 40-char hex SHA (e.g., `actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4`)
- **PASS:** Local actions (`./.github/actions/...`)
- **CRITICAL:** Tag or branch reference (e.g., `actions/checkout@v4`, `some-org/action@main`)

The `actions/` org actions must still be pinned — no namespace is inherently safe.

#### 1.2: Dangerous Workflow Triggers

**Applies to:** `github-actions`

1. **`pull_request_target`** — **CRITICAL** if workflow checks out PR head (`ref: ${{ github.event.pull_request.head.sha }}` or `ref: ${{ github.head_ref }}`), **HIGH** otherwise.
2. **`issue_comment`** — **HIGH** if no `author_association` permission check found.
3. **`workflow_run`** — **MEDIUM** if it processes artifacts without validation.
4. **`workflow_dispatch`** with `${{ github.event.inputs.* }}` interpolated in `run:` steps — **MEDIUM** (command injection).

#### 1.3: Docker Image Pinning

**Applies to:** `docker`

Check `Dockerfile` and `docker-compose.yml`/`docker-compose.yaml`:
- `FROM image:latest` — **HIGH**
- `FROM image:tag` without `@sha256:` digest — **MEDIUM**
- `FROM image@sha256:...` — **PASS**

#### 1.4: CI Lock File Usage

**Applies to:** `github-actions` AND (`js` OR `python` OR `rust`)

Scan all `.github/workflows/*.yml` and `.github/workflows/*.yaml` `run:` steps for dependency install commands that bypass lockfiles:

- `npm install` or `npm i` without using `ci` sub-command — **HIGH**. Fix: replace with `npm ci`.
- `yarn install` without `--frozen-lockfile` or `--immutable` — **HIGH**. Fix: add `--frozen-lockfile` (Yarn v1) or `--immutable` (Yarn v2+).
- `pnpm install` without `--frozen-lockfile` — **HIGH**. Fix: add `--frozen-lockfile`.
- `uv sync` without `--frozen` — **MEDIUM**. Fix: add `--frozen`.
- `cargo install` without `--locked` — **MEDIUM**. Fix: add `--locked`.

#### 1.5: Unpinned Dependencies in CI Commands

**Applies to:** `github-actions`

Scan `run:` steps in workflow files for commands that pull and execute unpinned packages at CI time:

- `npx <package>@latest` or `npx <package>` without version pin — **HIGH**. Fix: pin exact version (e.g., `npx package@1.2.3`). Recommend adding the tool as a `devDependency` and running via lockfile instead, since pinning the tool version does not pin its transitive dependencies.
- `pip install <package>` without `==` version pin in a `run:` step — **HIGH**. Fix: pin with `==` (e.g., `pip install package==1.2.3`). Recommend adding to `requirements.txt` with hashes (`--require-hashes`) or using `uv` with a lockfile, since pinning does not pin transitive dependencies.
- `npm install -g <package>` without version pin — **HIGH**. Fix: pin exact version (e.g., `npm install -g package@1.2.3`).
- `go install <package>@latest` — **HIGH**. Fix: pin to an exact version (e.g., `go install package@v1.2.3`). Treat bare `go install <package>` (no `@version` suffix) as a finding only when it runs outside the repo module context; in module-aware mode a bare `go install` resolves from the current module's `go.mod`/`go.sum` and is not unpinned. Note: `go install` always builds from source with the module's `go.sum`, so transitive dependencies are already verified.
- `curl ... | sh` or `wget ... | sh` (piped installs) — **CRITICAL**. Fix: replace with a package manager, or at minimum download to a file, verify a checksum, then execute.

---

### Category 2: Automatic Dependency Updates — recommendations only

Findings in this category **MUST NOT be fixed with code changes**. Include them as actionable recommendations in the PR description.

#### 2.1: Dependabot Cooldown ([docs](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-))

**Applies to:** `dependabot` OR `dependabot-pr-only`

If `dependabot-pr-only`: **HIGH** — no config file found, recommend creating `.github/dependabot.yml` with [`cooldown`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-) setting.

If `dependabot`: check each `updates` entry for [`cooldown.default-days`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-). **HIGH** if missing, **MEDIUM** if < 3 days.

#### 2.2: Renovate Minimum Release Age

**Applies to:** `renovate` OR `renovate-pr-only`

If `renovate-pr-only`: **HIGH** — no config file found, recommend creating `renovate.json` with [`minimumReleaseAge`](https://docs.renovatebot.com/configuration-options/#minimumreleaseage) and [`helpers:pinGitHubActionDigests`](https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests) preset.

If `renovate`: check config files for:
1. **[`minimumReleaseAge`](https://docs.renovatebot.com/configuration-options/#minimumreleaseage)** (top-level or in `packageRules`) — **HIGH** if missing, **MEDIUM** if < 3 days.
2. **[`helpers:pinGitHubActionDigests`](https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests)** in `extends` — **MEDIUM** if missing (when `github-actions` also detected).

---

### Category 3: Dev Dependency Install — recommendations only

Findings in this category **MUST NOT be fixed with code changes**. Include them as actionable recommendations in the PR description.

#### 3.1: Dependency Version Pinning

**Applies to:** all repo types with dependencies

**Exception — downgrade to PASS** when **both**: (1) a lockfile is committed, AND (2) a minimum release age / cooldown is configured (Dependabot [`cooldown`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-), Renovate [`minimumReleaseAge`](https://docs.renovatebot.com/configuration-options/#minimumreleaseage), pnpm [`minimumReleaseAge`](https://pnpm.io/settings#minimumreleaseage), or uv [`exclude-newer`](https://docs.astral.sh/uv/reference/settings/#exclude-newer)). If only one condition is met, flag as normal.

**JS** — check all `package.json` files (root + workspace packages). Flag `^`, `~`, `>=`, `>`, `*`, `latest` in `dependencies` (HIGH) and `devDependencies` (MEDIUM). Do NOT flag `workspace:*`/`workspace:^`/`workspace:~` references.

**Python** — check `pyproject.toml`, `requirements.txt`, `setup.py`, `setup.cfg`. Flag anything without `==` pin. HIGH.

**Rust** — check `Cargo.toml`. Flag deps without `=` prefix or using `*`. MEDIUM if `Cargo.lock` committed, HIGH if not.

**Go** — flag if `go.sum` not committed. HIGH.

#### 3.2: pnpm Supply Chain Protections

**Applies to:** `js-pnpm`

Check `pnpm-workspace.yaml`:
1. **[`minimumReleaseAge`](https://pnpm.io/settings#minimumreleaseage)** — recommended `3` or `7` days. **HIGH** if missing.
2. **[`blockExoticSubdeps`](https://pnpm.io/settings#blockexoticsubdeps)** — **MEDIUM** if missing.

#### 3.3: uv Supply Chain Protections

**Applies to:** `python-uv`

Check `pyproject.toml` `[tool.uv]` and `uv.toml` for **[`exclude-newer`](https://docs.astral.sh/uv/reference/settings/#exclude-newer)** (RFC 3339 datetime). **MEDIUM** if missing. **LOW** if present but older than 90 days.

## Phase 3: Output Format

Produce a markdown report with:
- Header: repo name, date, detected ecosystems
- Summary table: severity counts (CRITICAL/HIGH/MEDIUM/LOW/PASS)
- **Category 1 — CI Issues (code fixes applied in this PR):** findings grouped by severity, each with: file path (line N), issue description, concrete fix applied
- **Category 2 — Automatic Dependency Updates (recommendations):** findings grouped by severity, each with: issue description, recommended action for the user to take
- **Category 3 — Dev Dependency Install (recommendations):** findings grouped by severity, each with: issue description, recommended action for the user to take
- Passed checks list
- Collapsible "Why This Matters" section with real-world supply chain attack examples (include links and CVEs):
  - GitHub Actions: Trivy, [KICS](https://www.wiz.io/blog/teampcp-attack-kics-github-action), and [LiteLLM](https://docs.litellm.ai/blog/security-update-march-2026) ([CVE-2026-33634](https://nvd.nist.gov/vuln/detail/CVE-2026-33634)), reviewdog ([CVE-2025-30154](https://nvd.nist.gov/vuln/detail/CVE-2025-30154)), tj-actions/changed-files ([CVE-2025-30066](https://nvd.nist.gov/vuln/detail/CVE-2025-30066))
  - Package registries: [Axios npm compromise impacting OpenAI](https://openai.com/index/axios-developer-tool-compromise/) (2026), [PyTorch torchtriton](https://pytorch.org/blog/compromised-nightly-dependency/) (2022), colors.js ([CVE-2021-23567](https://nvd.nist.gov/vuln/detail/CVE-2021-23567), 2022), [ua-parser-js](https://github.com/advisories/GHSA-pjwm-rvh2-c87w) (2021), [event-stream](https://github.com/advisories/GHSA-mh6f-8j2x-4483) (2018)
- Prioritized next steps, split into:
  - **Already fixed** — Category 1 items addressed in this PR
  - **Follow-up actions** — Category 2 and 3 items for the user to address

## Phase 4: PR Guidance

### Duplicate PR Check

Before creating a PR, check for existing supply-chain hardening PRs:

1. Run: `gh pr list --search "supply chain" --state open --limit 5`
2. Run: `gh pr list --search "supply chain" --state merged --limit 5`
3. If any open PR title contains "supply chain" or "supply-chain" — **do not create a new PR**. Inform the user and suggest updating the existing PR.
4. If a merged PR with a matching title exists from the last 7 days — warn the user and ask for confirmation before proceeding.

### PR Creation

Check for a PR template (`.github/PULL_REQUEST_TEMPLATE.md` or `.github/PULL_REQUEST_TEMPLATE/`) and follow it if present. Title: `security: Harden supply chain <area>`.

**Code changes (Category 1 only):** Create a branch and commit fixes for all Category 1 findings:
- Pin GitHub Actions to SHA digests (1.1)
- Fix dangerous workflow triggers (1.2)
- Pin Docker images to SHA digests (1.3)
- Add `--frozen-lockfile` / `--locked` / `ci` flags to CI install commands (1.4)
- Pin or remove unpinned CI dependency references (1.5)

**PR description structure:**

```
## Summary
Brief description of supply chain hardening changes.

## Changes Made (CI Fixes)
- List each code change with file path and what was fixed

## Recommendations — Automatic Dependency Updates
> These items require manual follow-up and are NOT included as code changes in this PR.

- [ ] [2.1] ...
- [ ] [2.2] ...

## Recommendations — Dev Dependency Install Protections
> These items require manual follow-up and are NOT included as code changes in this PR.

- [ ] [3.1] ...
- [ ] [3.2] ...
- [ ] [3.3] ...
```

Only include recommendation sections that have findings. Use checkboxes so users can track follow-up.

## Important Notes

- Never suggest removing lockfiles.
- When suggesting SHA pinning, look up the actual SHA using `gh api repos/{owner}/{repo}/git/ref/tags/{tag}`. **Do not invent SHAs.**
- For monorepos, check all `package.json` files, not just root.
- Always include exact file paths and line numbers in findings.
- When fixing CI install commands (1.4), preserve existing flags — only append the lockfile flag if missing.
