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

### A: Dependency Version Pinning

**Applies to:** all repo types with dependencies

**Exception — downgrade to PASS** when **both**: (1) a lockfile is committed, AND (2) a minimum release age / cooldown is configured (Dependabot `cooldown`, Renovate `minimumReleaseAge`, pnpm `minimum-release-age`, or uv `exclude-newer`). If only one condition is met, flag as normal.

**JS** — check all `package.json` files (root + workspace packages). Flag `^`, `~`, `>=`, `>`, `*`, `latest` in `dependencies` (HIGH) and `devDependencies` (MEDIUM). Do NOT flag `workspace:*`/`workspace:^`/`workspace:~` references.

**Python** — check `pyproject.toml`, `requirements.txt`, `setup.py`, `setup.cfg`. Flag anything without `==` pin. HIGH.

**Rust** — check `Cargo.toml`. Flag deps without `=` prefix or using `*`. MEDIUM if `Cargo.lock` committed, HIGH if not.

**Go** — flag if `go.sum` not committed. HIGH.

### B: GitHub Actions SHA Pinning

**Applies to:** `github-actions`

Search `.github/workflows/*.yml`, `.github/workflows/*.yaml`, and `.github/actions/*/action.yml` for `uses:` directives.
- **PASS:** 40-char hex SHA (e.g., `actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4`)
- **PASS:** Local actions (`./.github/actions/...`)
- **CRITICAL:** Tag or branch reference (e.g., `actions/checkout@v4`, `some-org/action@main`)

The `actions/` org actions must still be pinned — no namespace is inherently safe.

### C: Dangerous Workflow Triggers

**Applies to:** `github-actions`

1. **`pull_request_target`** — **CRITICAL** if workflow checks out PR head (`ref: ${{ github.event.pull_request.head.sha }}` or `ref: ${{ github.head_ref }}`), **HIGH** otherwise.
2. **`issue_comment`** — **HIGH** if no `author_association` permission check found.
3. **`workflow_run`** — **MEDIUM** if it processes artifacts without validation.
4. **`workflow_dispatch`** with `${{ github.event.inputs.* }}` interpolated in `run:` steps — **MEDIUM** (command injection).

### D: pnpm Supply Chain Protections

**Applies to:** `js-pnpm`

Check `.npmrc` and `package.json` under `pnpm`:
1. **`minimum-release-age`** (`.npmrc`) / **`minimumReleaseAge`** (`package.json`) — recommended `3` or `7` days. **HIGH** if missing.
2. **`block-exotic-subdeps`** (`.npmrc`) / **`blockExoticSubdeps`** (`package.json`) — **MEDIUM** if missing.

### E: uv Supply Chain Protections

**Applies to:** `python-uv`

Check `pyproject.toml` `[tool.uv]` and `uv.toml` for **`exclude-newer`** (RFC 3339 datetime). **MEDIUM** if missing. **LOW** if present but older than 90 days.

### F: Dependabot Cooldown

**Applies to:** `dependabot` OR `dependabot-pr-only`

If `dependabot-pr-only`: **HIGH** — no config file found, recommend creating `.github/dependabot.yml` with [`cooldown`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-) setting.

If `dependabot`: check each `updates` entry for [`cooldown.default`](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-). **HIGH** if missing, **MEDIUM** if < 3 days.

### G: Renovate Minimum Release Age

**Applies to:** `renovate` OR `renovate-pr-only`

If `renovate-pr-only`: **HIGH** — no config file found, recommend creating `renovate.json` with [`minimumReleaseAge`](https://docs.renovatebot.com/configuration-options/#minimumreleaseage) and [`helpers:pinGitHubActionDigests`](https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests) preset.

If `renovate`: check config files for:
1. **[`minimumReleaseAge`](https://docs.renovatebot.com/configuration-options/#minimumreleaseage)** (top-level or in `packageRules`) — **HIGH** if missing, **MEDIUM** if < 3 days.
2. **[`helpers:pinGitHubActionDigests`](https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests)** in `extends` — **MEDIUM** if missing (when `github-actions` also detected).

### H: Docker Image Pinning

**Applies to:** `docker`

Check `Dockerfile` and `docker-compose.yml`/`docker-compose.yaml`:
- `FROM image:latest` — **HIGH**
- `FROM image:tag` without `@sha256:` digest — **MEDIUM**
- `FROM image@sha256:...` — **PASS**

## Phase 3: Output Format

Produce a markdown report with:
- Header: repo name, date, detected ecosystems
- Summary table: severity counts (CRITICAL/HIGH/MEDIUM/LOW/PASS)
- Findings grouped by severity, each with: file path (line N), issue description, concrete fix
- Passed checks list
- Collapsible "Why This Matters" section with real-world supply chain attack examples (include links and CVEs):
  - GitHub Actions: tj-actions/changed-files (CVE-2025-30066), reviewdog (CVE-2025-30154), Trivy (CVE-2026-33634), KICS, LiteLLM
  - Package registries: event-stream (2018), ua-parser-js (2021), colors.js (2022), PyTorch torchtriton (2022)
- Prioritized recommended next steps

## Phase 4: PR Guidance

When creating a fix PR: check for a PR template (`.github/PULL_REQUEST_TEMPLATE.md` or `.github/PULL_REQUEST_TEMPLATE/`) and follow it if present. Title: `security: Harden supply chain <area>`.

## Important Notes

- Never suggest removing lockfiles.
- When suggesting SHA pinning, look up the actual SHA using `gh api repos/{owner}/{repo}/git/ref/tags/{tag}`. **Do not invent SHAs.**
- For monorepos, check all `package.json` files, not just root.
- Always include exact file paths and line numbers in findings.
