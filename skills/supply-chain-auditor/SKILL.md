---
name: supply-chain-auditor
description: Audit a repository for supply chain security risks. Use when the user asks to "audit supply chain", "check supply chain security", "audit dependencies", "check dependency pinning", "review GitHub Actions security", "check for supply chain attacks", "dependency security audit", "audit CI/CD security", "check for unpinned actions", "harden supply chain", or mentions supply chain security, dependency hygiene, or CI/CD hardening. Activate even for partial matches like "are my deps pinned?" or "is my CI safe?".
---

# Supply Chain Security Auditor

Audit any repository for supply chain security risks. Detect the repo type, run category-specific checks, and produce an actionable report with severity ratings and real-world attack context.

## Phase 1: Repo Detection

Before running any checks, detect the repository characteristics. Use Glob and Grep. Record which categories apply — skip inapplicable checks silently.

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

**Fallback detection via PR history:** Some repos use Dependabot or Renovate with default configuration (enabled via GitHub settings UI) without committing a config file. To catch these:

- If `dependabot` was **not** detected from the table above, run:
  ```
  gh pr list --author 'app/dependabot' --state all --limit 1
  ```
  If at least one PR is returned, set category flag `dependabot-pr-only`.

- If `renovate` was **not** detected from the table above, run:
  ```
  gh pr list --author 'app/renovate' --state all --limit 1
  ```
  If at least one PR is returned, set category flag `renovate-pr-only`.

Use `--state all` to catch both open and closed/merged PRs. Use `--limit 1` for efficiency — only one PR is needed to confirm the tool is in use. Only run these `gh` commands when the corresponding config-file detection found nothing.

## Phase 2: Audit Checks

Run every applicable check. Assign each finding a severity:

- **CRITICAL** — Actively exploitable, matches known attack patterns
- **HIGH** — Strong risk, one step from exploitation
- **MEDIUM** — Defense-in-depth gap, opportunistic risk
- **LOW** — Best practice missing, minimal direct risk
- **PASS** — Check passed, no issue found

### A: Dependency Version Pinning

**Applies to:** all repo types with dependencies

Every dependency should specify an exact version, not a range. Ranges allow silent adoption of compromised releases.

**However**, version ranges are acceptable when **both** of these conditions are met:
1. A lockfile is committed (`pnpm-lock.yaml`, `yarn.lock`, `package-lock.json` for JS; `uv.lock` for Python) — ensures reproducible installs
2. A minimum release age / cooldown is configured (Dependabot `cooldown`, Renovate `minimumReleaseAge`, pnpm `minimum-release-age`, or uv `exclude-newer`) — prevents auto-adopting freshly published compromised versions

When both conditions are met, **downgrade** range findings to **PASS** for that ecosystem — the lockfile pins the actual resolved version and the release age gate blocks rapid exploitation. If only one condition is met, flag as normal.

**JS repos** — check all `package.json` files (root + workspace packages):
- Flag versions starting with `^`, `~`, `>=`, `>`, `*`, or `latest` in `dependencies` and `devDependencies`
- Exact versions look like `"1.2.3"` (no prefix)
- **Do NOT flag** pnpm/yarn workspace protocol references (`workspace:*`, `workspace:^`, etc.) — these resolve internally
- **Do NOT flag** if a lockfile (`pnpm-lock.yaml`, `yarn.lock`, or `package-lock.json`) is committed **and** a release age gate is configured (see checks D, F, G)
- Severity: HIGH for `dependencies`, MEDIUM for `devDependencies`

**Python repos** — check `pyproject.toml`, `requirements.txt`, `setup.py`, `setup.cfg`:
- Flag `>=`, `~=`, `>`, `!=`, `*`, or bare package names without `==`
- Exact pins: `package==1.2.3`
- **Do NOT flag** if `uv.lock` is committed **and** `exclude-newer` is configured (see check E)
- Severity: HIGH

**Rust repos** — check `Cargo.toml`:
- Flag any dependency without `=` prefix or using `*`
- Exact pins: `=1.2.3`
- MEDIUM if `Cargo.lock` is committed, HIGH if not

**Go repos** — check `go.mod`:
- Go pins by default via `go.sum`. Flag if `go.sum` is not committed.
- Severity: HIGH if `go.sum` missing

### B: GitHub Actions SHA Pinning

**Applies to:** `github-actions`

All third-party actions must be pinned to a full 40-character commit SHA, not a tag or branch. Tags are mutable and can be force-pushed to point at malicious code — this is exactly how the tj-actions/changed-files and Trivy attacks worked.

Search `.github/workflows/*.yml`, `.github/workflows/*.yaml`, and `.github/actions/*/action.yml`:
- Find all `uses:` directives
- **PASS:** `uses: actions/checkout@a5ac7e51b41094c92402da3b24376905380afc29 # v4` (40-char hex SHA)
- **PASS:** `uses: ./.github/actions/local-action` (local actions are fine)
- **CRITICAL:** `uses: actions/checkout@v4` (mutable tag)
- **CRITICAL:** `uses: some-org/some-action@main` (mutable branch)

A comment after the SHA showing the tag is fine and encouraged for readability.

**Renovate auto-pinning:** If `renovate` or `renovate-pr-only` is detected, recommend adding the [`helpers:pinGitHubActionDigests`](https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests) preset to the Renovate config. This makes Renovate automatically pin GitHub Actions to SHA digests and keep them updated:
```json
{
  "extends": ["helpers:pinGitHubActionDigests"]
}
```

### C: Dangerous Workflow Triggers

**Applies to:** `github-actions`

Search all `.github/workflows/*.yml` and `.github/workflows/*.yaml` for:

1. **`pull_request_target`** — Runs in the context of the base branch with write permissions and access to secrets. If the workflow checks out the PR head (`ref: ${{ github.event.pull_request.head.sha }}` or `ref: ${{ github.head_ref }}`), an attacker can submit a PR with malicious code that executes with repo secrets. **CRITICAL** if checkout of PR head is detected, **HIGH** otherwise.

2. **`issue_comment`** — Runs with write access. Often used for `/deploy`-style comment triggers. Check if the workflow validates the commenter's permissions (`author_association`). **HIGH** if no permission check found.

3. **`workflow_run`** — Can inherit artifacts from untrusted workflow runs. **MEDIUM** if it processes artifacts without validation.

4. **`workflow_dispatch` with inputs in `run:` steps** — If user-supplied inputs are interpolated directly into shell commands (`${{ github.event.inputs.* }}`), this is a command injection vector. **MEDIUM**.

### D: pnpm Supply Chain Protections

**Applies to:** `js-pnpm`

Check `.npmrc` and `package.json` `pnpm` config for:

1. **`minimum-release-age`** (`.npmrc`) or **`minimumReleaseAge`** (`package.json` under `pnpm`) — Reject packages published less than N days ago. Recommended: `3` or `7` days. This would have caught event-stream, ua-parser-js, and colors.js since the malicious versions were exploited within hours.
   - **HIGH** if missing

2. **`block-exotic-subdeps`** (`.npmrc`) or **`blockExoticSubdeps`** (`package.json` under `pnpm`) — Blocks subdependencies using install scripts, native builds, or exotic protocols. Prevents transitive attacks.
   - **MEDIUM** if missing

### E: uv Supply Chain Protections

**Applies to:** `python-uv`

Check `pyproject.toml` under `[tool.uv]` and `uv.toml` for:

1. **`exclude-newer`** — An RFC 3339 datetime string. uv ignores any package version published after this date. Protects against future compromises.
   - **MEDIUM** if missing
   - If present but older than 90 days: **LOW** (stale — needs updating)

Reference: https://docs.astral.sh/uv/reference/settings/#exclude-newer

### F: Dependabot Cooldown

**Applies to:** `dependabot` OR `dependabot-pr-only`

**If `dependabot-pr-only`** (detected via PR history, no config file):
- Report that Dependabot is actively used in this repository (PRs exist) but no `.github/dependabot.yml` configuration file was found. This means Dependabot is running with fully default settings — no cooldown period, no custom grouping, no version filtering.
- **HIGH** severity
- **Fix:** Create a `.github/dependabot.yml` file with explicit configuration including a `cooldown` setting.

**If `dependabot`** (config file exists):

Check `.github/dependabot.yml` for `cooldown` within each `updates` entry:

```yaml
updates:
  - package-ecosystem: "npm"
    cooldown:
      default: 3
```

- **HIGH** if `cooldown` is missing entirely
- **MEDIUM** if `default` is less than 3 days

Reference: https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown-

### G: Renovate Minimum Release Age

**Applies to:** `renovate` OR `renovate-pr-only`

**If `renovate-pr-only`** (detected via PR history, no config file):
- Report that Renovate is actively used in this repository (PRs exist) but no Renovate configuration file was found (`renovate.json`, `renovate.json5`, `.renovaterc`, `.renovaterc.json`, or `renovate` key in `package.json`). This means Renovate is running with fully default preset — no minimum release age, no custom package rules.
- **HIGH** severity
- **Fix:** Create a `renovate.json` file with explicit configuration including `minimumReleaseAge` and the [`helpers:pinGitHubActionDigests`](https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests) preset for automatic GitHub Actions SHA pinning.

**If `renovate`** (config file exists):

Check `renovate.json`, `renovate.json5`, `.renovaterc`, `.renovaterc.json`, or `renovate` key in `package.json` for:

1. **`minimumReleaseAge`** at the top level or within `packageRules` — e.g., `"3 days"` or `"7 days"`
   - **HIGH** if missing
   - **MEDIUM** if less than 3 days

2. **`helpers:pinGitHubActionDigests`** in the `extends` array — automatically pins GitHub Actions to SHA digests and keeps them updated
   - **MEDIUM** if missing (when `github-actions` category is also detected)

Reference: https://docs.renovatebot.com/key-concepts/minimum-release-age/ , https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests

### H: Docker Image Pinning

**Applies to:** `docker`

Check `Dockerfile` and `docker-compose.yml`/`docker-compose.yaml` for:
- `FROM image:latest` — **HIGH**
- `FROM image:tag` without `@sha256:` digest — **MEDIUM**
- `FROM image@sha256:abcdef...` — **PASS**

## Phase 3: Output Format

Produce the report in this structure:

```markdown
# Supply Chain Security Audit

**Repository:** <repo name>
**Date:** <current date>
**Detected ecosystem(s):** <comma-separated list of detected categories>

## Summary

| Severity | Count |
| -------- | ----- |
| CRITICAL | N     |
| HIGH     | N     |
| MEDIUM   | N     |
| LOW      | N     |
| PASS     | N     |

## Findings

### CRITICAL

#### [C-1] <Short title>
- **File:** `path/to/file` (line N)
- **Issue:** <specific description>
- **Fix:** <concrete remediation with exact code/config change>

### HIGH
...

### MEDIUM
...

### LOW
...

### Passed Checks
- <Brief list of checks that passed>

## Why This Matters

<details>
<summary>Supply chain attacks via GitHub Actions (March 2025–2026)</summary>

- **[tj-actions/changed-files](https://github.com/tj-actions/changed-files/security/advisories/GHSA-mcph-m25j-8gfq) (Mar 2025) — [CVE-2025-30066](https://nvd.nist.gov/vuln/detail/CVE-2025-30066) (CVSS 8.6)** — Compromised GitHub Action that dumped CI secrets to workflow logs; affected thousands of repositories trusting tag-based references.

- **[reviewdog](https://github.com/reviewdog/reviewdog/issues/2079) (Mar 2025) — [CVE-2025-30154](https://nvd.nist.gov/vuln/detail/CVE-2025-30154)** — Multiple reviewdog GitHub Actions compromised via stolen PAT; malicious code injected into tagged releases, exfiltrating secrets from CI runners.

- **[Trivy](https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23) (Mar 19–22, 2026) — [CVE-2026-33634](https://www.tenable.com/cve/CVE-2026-33634) (CVSS 9.4)** — Misconfigured `pull_request_target` workflow exposed org-scoped PAT; incomplete credential rotation after first breach let attackers return and poison 76 action tags.

- **[KICS GitHub Action](https://www.wiz.io/blog/teampcp-attack-kics-github-action) (Mar 23, 2026) — no dedicated CVE** — Service account (`cx-plugins-releases`) compromised, likely via credentials stolen from Trivy victims' CI pipelines; tracked under CVE-2026-33634 / Checkmarx's own [security update](https://checkmarx.com/blog/checkmarx-security-update/).

- **[LiteLLM](https://docs.litellm.ai/blog/security-update-march-2026) (Mar 24, 2026) — no dedicated CVE** — PyPI publish credentials exfiltrated from LiteLLM's own CI pipeline, which was running the compromised Trivy action; malicious versions then published directly to PyPI.

</details>

<details>
<summary>Supply chain attacks via package registries (2018–2022)</summary>

- **[event-stream](https://blog.npmjs.org/post/180565383195/details-about-the-event-stream-incident) (2018)** — Maintainer social-engineered into transferring npm package; attacker added dependency that stole cryptocurrency wallet keys.

- **[ua-parser-js](https://github.com/nicedoc/ua-parser-js/issues/536) (Oct 2021)** — Compromised npm package installed cryptominers and credential stealers; 8M+ weekly downloads affected.

- **[colors.js / faker.js](https://www.bleepingcomputer.com/news/security/dev-corrupts-npm-libs-colors-and-faker-breaking-thousands-of-apps/) (Jan 2022)** — Maintainer deliberately published infinite-loop versions, breaking thousands of downstream projects.

- **[PyTorch nightly](https://pytorch.org/blog/compromised-nightly-dependency/) (Dec 2022)** — Dependency confusion attack via `torchtriton` package on PyPI that exfiltrated system data.

</details>

## Recommended Next Steps

1. <Prioritized list of concrete actions based on findings>
```

## Phase 4: PR Guidance

When creating a PR to fix supply chain issues found by this audit:

1. Check for a PR template: `.github/PULL_REQUEST_TEMPLATE.md` or `.github/PULL_REQUEST_TEMPLATE/`
2. If a PR template exists, **read it and follow its structure exactly**, filling in all sections
3. Use the audit findings as the basis for the PR description
4. Title format: `security: Harden supply chain <specific area>`

## Important Notes

- Never suggest removing lockfiles — they are critical for reproducibility.
- When suggesting SHA pinning for GitHub Actions, look up the actual current SHA for the tag using `gh api repos/{owner}/{repo}/git/ref/tags/{tag}` or the workflow file's existing comment. Do not invent SHAs.
- For monorepos, check all `package.json` files, not just root.
- Do NOT flag pnpm/yarn workspace protocol references (`workspace:*`, `workspace:^`, `workspace:~`).
- The `actions/` org actions (checkout, setup-node, etc.) must still be pinned to SHAs despite being GitHub-official. The tj-actions attack proved no namespace is inherently safe.
- Always include exact file paths and line numbers in findings.
