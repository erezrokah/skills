---
name: you-dont-need-isr
description: Audit a Next.js project's Incremental Static Regeneration usage and suggest simpler alternatives where ISR is overkill. Use when the user asks to evaluate ISR, check if ISR is necessary, simplify a Next.js caching strategy, or wants to know "do I actually need ISR?". Activate for partial matches like "should I be using ISR?", "ISR or SSG?", "is revalidate worth it?", or "my Next.js caching is confusing".
---

# You Don't Need ISR

Audit a Next.js project's Incremental Static Regeneration (ISR) usage. For each occurrence, classify whether ISR is genuinely the right tool or whether a simpler primitive (SSG + webhook revalidation, SSR + CDN cache headers, on-demand revalidation, client-side fetch, or a beefier build runner) does the same job with less platform lock-in and less complexity. Then walk the user through fixes interactively.

This skill audits, explains, and assists. It is **not** a refactor bot — bias toward fewer, higher-confidence changes and never auto-fix on uncertainty.

## Phase 1: Project Detection

Detect characteristics using Glob and Grep. Skip inapplicable phases silently.

| Signal | Flag |
| --- | --- |
| `next.config.{js,mjs,ts}` exists | `next-app` |
| `package.json` has `next` dep | `next-app` (record version) |
| `app/` directory present | `app-router` |
| `pages/` directory present | `pages-router` |
| `vercel.json` OR `.vercel/` | `vercel-deploy` |
| Vercel MCP tools listed in available tools | `vercel-mcp-available` |
| `.github/workflows/*.{yml,yaml}` with a `next build` / `npm run build` / `pnpm build` / `yarn build` / `bun run build` step | `gha-build` (record `runs-on:` value verbatim) |
| `vercel-deploy` set AND no `gha-build` step | `vercel-build` (record build-machine tier from Vercel project settings if MCP available — Hobby / Pro Standard / Pro Enhanced / Enterprise Turbo) |

If `next-app` is not detected, stop with a one-line explanation that this skill only audits Next.js projects.

## Phase 2: ISR Usage Discovery

For every detected router, grep for the patterns below and record `{file, line, mechanism, window, tags}` for every hit.

**App Router** (`app/**/{page,layout,route}.{ts,tsx,js,jsx}`):
- `export const revalidate = N`
- `fetch(..., { next: { revalidate: N } })`
- `fetch(..., { next: { tags: [...] } })`
- `unstable_cache` imports + call sites
- `revalidatePath(...)` / `revalidateTag(...)` (server actions, route handlers)
- `generateStaticParams` (context for dynamic ISR routes)
- `export const dynamic = 'force-static' | 'force-dynamic' | 'auto'`
- `export const dynamicParams = true | false`

**Pages Router** (`pages/**/*.{ts,tsx,js,jsx}`):
- `getStaticProps` returning `{ revalidate: N }`
- `getStaticPaths` with `fallback: 'blocking' | true`
- API routes calling `res.revalidate(...)` (on-demand ISR)

Also record nearby signals that change the verdict:
- Whether the route reads `cookies()`, `headers()`, `searchParams`, or auth — implies request-scoped data, ISR cannot personalize.
- Whether `revalidatePath`/`revalidateTag` is called anywhere (a wired-up tag system is a strong NECESSARY signal).
- Whether the project has a CMS webhook → CI rebuild flow already.

## Phase 3: Build & Runtime Data Pull (best-effort)

### Vercel data
If `vercel-mcp-available` AND the project is linked:
- Recent build durations (median, p95).
- ISR / function invocation counts for the discovered routes.
- Bandwidth from prerendered pages (signal of "ISR is doing real work").

### GitHub Actions data
If `gha-build` is set:
- Pull recent run durations: `gh run list --workflow=<file> --json databaseId,createdAt,conclusion,startedAt,updatedAt --limit 50`. Compute median build duration and monthly run count.
- Classify `runs-on`:
  - `ubuntu-latest` / `ubuntu-22.04` / `ubuntu-24.04` → **GitHub-hosted standard** (2 vCPU, 7 GB RAM, billed per minute on private repos).
  - `ubicloud-*` → **Ubicloud already in use** — note the tier and skip the swap suggestion.
  - Anything else → record verbatim, do not classify further.

### Build cost projection
Run whenever build-pressure ISR is suspected. **Pricing must come from a live WebFetch — never hardcode $/min, included minutes, or vCPU counts.**

1. **WebFetch Ubicloud GitHub Actions runner specs + pricing** from `https://www.ubicloud.com/use-cases/github-actions` and the linked pricing page. Parse at least `standard-2`, `standard-4`, `standard-8`, `standard-16` (vCPU, RAM, $/min for x64 Linux).
2. **WebFetch Vercel build-machine specs + pricing** from `https://vercel.com/docs/builds/managing-builds#build-container-resources` and `https://vercel.com/pricing`. Parse vCPU, RAM, included minutes, and overage $/min for each build-container tier (Hobby / Pro Standard / Pro Enhanced / Enterprise Turbo).
3. **Project per scenario.** Bracket every estimate — Next.js builds are not perfectly parallel:
   - **Current setup**: median build × $/min × monthly run count.
   - **Each Ubicloud tier with ≥ 2× the current vCPU**: estimated build time = `current_minutes × max(1.5, current_vCPU / target_vCPU)` (best case ≈ vCPU ratio, worst case ≈ 1.5×). Compute estimated $/build and $/month at the same monthly run count.
   - **For `vercel-build` projects**: also project "stay on Vercel, upgrade build-container tier" using the parsed Vercel pricing, **and** "migrate build to GHA + Ubicloud `standard-8`" — flag the migration variant as an architectural shift, not a config swap.
4. **Output**: a comparison block (current vs each candidate) for Phase 6's report. Always cite the source URL and the date pulled.

If WebFetch fails or pages can't be parsed, emit `pricing unavailable — see <URL>` placeholders. Never guess. If neither Vercel data nor GHA data is available, skip silently and note "code-patterns only" in the report. **Never block on missing data.**

## Phase 4: Verdict per ISR Usage

Classify each occurrence found in Phase 2. If uncertain, default to **MAYBE** — never auto-fix on a coin flip.

Verdict levels: **NECESSARY** · **MAYBE** · **OVERKILL** · **WRONG-TOOL**

### OVERKILL
- `revalidate ≥ 86400` (≥ 1 day) AND build duration < 10 min — use SSG + webhook rebuild instead.
- `revalidate` window ≫ build duration AND total ISR routes < ~500.
- `unstable_cache` wrapping a pure / deterministic function (no I/O).
- Route fetches no remote data per request — ISR added "just in case".
- **Build-pressure ISR on a small runner**: ISR exists purely to dodge build time, the build runs on `ubuntu-latest` (2 vCPU), and the Phase 3 projection shows an Ubicloud tier that meaningfully shortens the build at comparable or lower $/month. Suggest the Ubicloud swap **before** suggesting any architectural change. Quote the projected build time and monthly $ delta in the verdict.

### WRONG-TOOL
- Page reads `cookies()` / `headers()` / auth — ISR cannot personalize; use SSR.
- `revalidate < 60` — sub-minute windows aren't reliably honored across regions; use SSR + short `s-maxage`.
- `getStaticPaths` with `fallback: true` / `'blocking'` for an unbounded set that should be SSR.

### NECESSARY
- Hundreds/thousands of routes whose full SSG build duration exceeds the team's tolerable CI window even on a beefy runner.
- Tag-based invalidation actually wired up (`revalidateTag` called from server actions / webhooks).
- Long stale-while-revalidate windows the team explicitly relies on for "never block the user, never go down".

### MAYBE
Anything else — surface the alternatives and let the user decide.

## Phase 5: Reasons → Alternatives Reference

This table ships in every report.

| Reason for reaching for ISR | When ISR is actually needed | Simpler alternative |
| --- | --- | --- |
| "Content updates every N hours" | Many pages AND slow build | SSG + webhook revalidation, or SSR + `Cache-Control: s-maxage=N, stale-while-revalidate=...` |
| "Build time too long" | > 30 min on a beefy runner AND not parallelizable | **First**: move the GHA build to an Ubicloud runner (e.g. `ubicloud-standard-8`) — Phase 3 surfaces a concrete time/$ projection from live Ubicloud + Vercel pricing. **Then**: partial builds, or move long-tail routes to SSR. |
| "SEO with dynamic data" | Rare | SSR + CDN cache — bots see identical HTML |
| "Reduce server cost / load" | Huge traffic on cacheable routes | CDN cache of SSR responses |
| "Update without redeploy" | Many updates per hour | On-demand revalidation, or webhook → CI rebuild |
| "Time-sensitive prices/inventory" | ~minute precision and stale-OK | SSR + short CDN cache; client fetch for the volatile bits |
| "Personalized content" | Never | SSR or client-side fetch |

### Why each alternative works

Print this section verbatim in the report so the user has the reasoning alongside the verdicts.

**Content updates every N hours → SSG + webhook, or SSR + CDN headers.**
ISR's mechanic here is "background-regenerate on a timer; serve cached HTML in the meantime". A webhook-triggered rebuild is exact rather than eventually consistent — cost is one CI run per actual update, not N regenerations across all routes. SSR + `s-maxage=N, stale-while-revalidate=...` produces the same user-visible behavior (instant response, freshness window of N) using plain HTTP cache semantics that work on any CDN, not just Vercel's ISR cache. ISR genuinely wins only when you need **both** timer freshness and persistence across deploys, on a route set too large to rebuild.

**Build time too long → faster runner first, then partial builds, then SSR.**
ISR is often a workaround for "our build is slow on `ubuntu-latest`" — but that solves the wrong problem. A faster runner is a one-line config change with zero architectural risk: a `next build` is mostly CPU-bound, so an 8-vCPU machine often cuts time 2–4× without touching app code. Partial builds (Turbopack, Nx affected, monorepo-aware caching) come next — moderate effort, keeps SSG semantics. Moving the long tail to SSR is the biggest behavior change (cold starts, runtime cost), so it's the alternative of last resort, not the first reach.

**SEO with dynamic data → SSR + CDN cache.**
Crawlers parse HTML and don't care whether it was generated at build, regenerated by ISR, or rendered by SSR — same bytes either way. The only thing that matters for SEO is that fully-rendered HTML reaches them quickly, which SSR behind a CDN delivers just as well. ISR is solving a problem that doesn't exist for crawlers.

**Reduce server cost / load → CDN cache of SSR.**
ISR's cost benefit is "one regeneration serves many requests". A CDN does the same thing for SSR responses — one origin hit fills the cache, subsequent requests serve from edge memory at zero compute. You also get more control: per-route TTLs, surrogate-key purges, geographic distribution. ISR's only edge is platform-managed cache storage; if you already pay for a CDN, you have that.

**Update without redeploy → on-demand revalidation or webhook → CI rebuild.**
This is the closest ISR feature to "no alternative needed". If you update frequently (>1×/hour) across many routes, on-demand revalidation (`revalidatePath`/`revalidateTag`) is exactly the right tool. The alternative kicks in when updates are rare: a webhook from your CMS that triggers a CI build is dramatically simpler — no cache invalidation logic, no tag bookkeeping, no stale-cache debugging. Pick based on cadence.

**Time-sensitive prices/inventory → SSR + short CDN cache, or client fetch for the volatile bits.**
ISR's failure mode here is sharp: any cached page is by definition stale, and on a price/inventory page "stale" can mean overselling or showing a wrong price. SSR + short CDN cache (`s-maxage=30`) bounds staleness and hits origin immediately on price changes you push. A static shell with client-side fetch for the volatile bits gives a fast, SEO-friendly first paint and reduces worst-case staleness to one user's network round-trip rather than "ISR window minus elapsed time".

**Personalized content → SSR or client-side fetch.**
ISR can't do this — full stop. ISR caches one HTML response per route and serves it to everyone, so the moment a page reads `cookies()`, session, user role, or anything request-scoped, the cache entry is wrong for every other user. SSR (which has access to the request) or client-side fetch (which runs after auth) are the only correct tools. If you see ISR on a personalized page, it's a bug, not a tradeoff — Phase 4 classifies it `WRONG-TOOL` rather than `OVERKILL` for that reason.

## Phase 6: Report Format

Produce a markdown report:

- **Header**: project name, date, detected router(s), Next.js version, deploy target.
- **Summary**: routes scanned, ISR occurrences, verdict breakdown (`NECESSARY` / `MAYBE` / `OVERKILL` / `WRONG-TOOL`).
- **Findings table**: one row per occurrence — `file:line`, mechanism, window, verdict, suggested alternative.
- **Build cost projection** (if produced in Phase 3): comparison block — current runner / build tier vs each Ubicloud candidate (and, for `vercel-build`, the upgrade-tier option) — showing estimated build time, $/build, $/month. Always cite the source URL and the date the pricing was pulled. Always include "rough estimate, actual will vary by build parallelism".
- **Vercel data** (collapsed `<details>` if pulled).
- **Reasons → Alternatives** table from Phase 5, **including the "Why each alternative works" section verbatim**.
- **Recommended next steps**, split into:
  - **Try first** — interactive fixes from Phase 7 (per-finding apply/skip).
  - **Investigate** — `MAYBE` findings the user should review.
  - **Leave alone** — `NECESSARY` findings (briefly explain why).

## Phase 7: Interactive Fixes

For each `OVERKILL` or `WRONG-TOOL` finding, in order:

1. Show file/line and current code.
2. Show the suggested replacement — concrete code, not vague advice. Common transforms:
   - `export const revalidate = N` → remove (pure SSG) and add a one-line note about wiring a webhook → CI rebuild.
   - `fetch(..., { next: { revalidate: N } })` → `fetch(...)` plus `Cache-Control: s-maxage=N, stale-while-revalidate=86400` headers via `route.ts` / middleware.
   - `getStaticProps + revalidate` → `getServerSideProps` plus `res.setHeader('Cache-Control', 's-maxage=N, stale-while-revalidate=86400')`.
   - `unstable_cache` around a pure function → drop the wrapper.
   - **Build-pressure ISR on small runner** → propose a concrete `runs-on:` swap in the relevant `.github/workflows/*.yml` (`ubuntu-latest` → `ubicloud-standard-8`, or whichever tier Phase 3 projected best). Quote the projected build time and $/month delta. One-line note that Ubicloud requires installing their GitHub App and is billed per minute. Apply only if the user agrees. **Also check the workflow for `.next/cache` caching** (an `actions/cache` step whose `path:` includes `.next/cache`) — if missing, propose adding it as a separate edit using the pattern from the migration sub-section below; without it the runner-swap savings get partially eaten by Next.js rebuilding from scratch every run.
3. Use `AskUserQuestion`: **Apply** / **Skip** / **Modify**.
4. If `Apply`: perform the edit. Batch all applied changes into a single draft PR at the end if the user opts in (PR title format: `feat: <Summary>` or `fix: <Summary>`, draft).

**Never auto-fix `NECESSARY` or `MAYBE`** — only surface them.

### Migration: Vercel-built project → GitHub Actions + Vercel deploy

Triggered only when the project is `vercel-build` AND the Phase 3 projection shows a meaningful win on GHA + Ubicloud AND the user explicitly agrees this is worth the architectural shift. Source: Vercel KB — [How can I use GitHub Actions with Vercel?](https://vercel.com/kb/guide/how-can-i-use-github-actions-with-vercel)

**Prerequisites the user must do themselves** (do not attempt to automate — these touch their Vercel account and GitHub repo secrets):

1. **Get the Vercel project linked locally** (if not already): run `vercel login`, then `vercel link` inside the project folder. This creates `.vercel/project.json` containing `projectId` and `orgId`.
2. **Create a Vercel access token** — see [Vercel Access Token guide](https://vercel.com/guides/how-do-i-use-a-vercel-api-access-token).
3. **Add three GitHub repo secrets** (`Settings → Secrets and variables → Actions`):
   - `VERCEL_TOKEN` — the access token from step 2.
   - `VERCEL_ORG_ID` — `orgId` from `.vercel/project.json`.
   - `VERCEL_PROJECT_ID` — `projectId` from `.vercel/project.json`.
4. **Disconnect Vercel's Git integration** for this project in the Vercel dashboard once the GHA workflows are confirmed working — otherwise both Vercel and GHA will deploy on every push and you'll get duplicates. Leave this **as a final step the user does manually**.

**Workflow files to add** (use the runner tier Phase 3 projected best — `ubicloud-standard-8` shown as the default; swap `runs-on:` to whichever tier the user picked).

These extend the Vercel KB examples with two cache layers per Next.js's [official CI guidance](https://nextjs.org/docs/app/guides/ci-build-caching):
- **Package-manager cache** via `actions/setup-node` (avoids re-downloading deps on every run). Pick `cache: npm` / `pnpm` / `yarn` / `bun` based on the lockfile detected in Phase 1.
- **`.next/cache`** via `actions/cache` so Next.js's incremental build cache survives between runs. **Vercel handles this automatically; once you move builds to GHA you must add it yourself or builds restart from scratch every time and eat the runner-swap savings.**

The cache key uses a two-tier `key` / `restore-keys` pattern so the cache is regenerated when source changes but reuses the previous deps cache as a starting point — also taken from the Next.js docs.

`.github/workflows/preview.yaml`:

```yaml
name: Vercel Preview Deployment
env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
on:
  push:
    branches-ignore:
      - main
jobs:
  Deploy-Preview:
    runs-on: ubicloud-standard-8
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - name: Cache Next.js build
        uses: actions/cache@v4
        with:
          path: |
            ~/.npm
            ${{ github.workspace }}/.next/cache
          key: ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.jsx', '**/*.ts', '**/*.tsx') }}
          restore-keys: |
            ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-
      - name: Install Vercel CLI
        run: npm install --global vercel@latest
      - name: Pull Vercel Environment Information
        run: vercel pull --yes --environment=preview --token=${{ secrets.VERCEL_TOKEN }}
      - name: Build Project Artifacts
        run: vercel build --token=${{ secrets.VERCEL_TOKEN }}
      - name: Deploy Project Artifacts to Vercel
        run: vercel deploy --prebuilt --token=${{ secrets.VERCEL_TOKEN }}
```

`.github/workflows/production.yaml`:

```yaml
name: Vercel Production Deployment
env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
on:
  push:
    branches:
      - main
jobs:
  Deploy-Production:
    runs-on: ubicloud-standard-8
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - name: Cache Next.js build
        uses: actions/cache@v4
        with:
          path: |
            ~/.npm
            ${{ github.workspace }}/.next/cache
          key: ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.jsx', '**/*.ts', '**/*.tsx') }}
          restore-keys: |
            ${{ runner.os }}-nextjs-${{ hashFiles('**/package-lock.json') }}-
      - name: Install Vercel CLI
        run: npm install --global vercel@latest
      - name: Pull Vercel Environment Information
        run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
      - name: Build Project Artifacts
        run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
      - name: Deploy Project Artifacts to Vercel
        run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

**Adapting the cache step to other package managers** (pick based on the lockfile detected in Phase 1):

| Lockfile | `setup-node` `cache:` | `actions/cache` `path:` (deps line) | `hashFiles` glob |
| --- | --- | --- | --- |
| `package-lock.json` | `npm` | `~/.npm` | `**/package-lock.json` |
| `pnpm-lock.yaml` | `pnpm` (also requires `pnpm/action-setup` step before `setup-node`) | `~/.local/share/pnpm/store` | `**/pnpm-lock.yaml` |
| `yarn.lock` | `yarn` | `~/.yarn/cache` (Berry) or `~/.cache/yarn` (Classic) | `**/yarn.lock` |
| `bun.lock` / `bun.lockb` | (use `oven-sh/setup-bun` instead of `setup-node`) | `~/.bun/install/cache` | `**/bun.lock` or `**/bun.lockb` |

**Preview vs production differences** (from the same KB):

| Aspect | Preview | Production |
| --- | --- | --- |
| Trigger | all branches except `main` | `main` only |
| `vercel pull` | `--environment=preview` | `--environment=production` |
| `vercel build` | no flag | `--prod` |
| `vercel deploy` | `--prebuilt` | `--prebuilt --prod` |

`vercel build` produces a `.vercel/output` folder; `vercel deploy --prebuilt` uploads it without rebuilding on Vercel — that's what shifts the build minutes onto your GHA runner instead of Vercel's build container.

**How the skill applies this**:
1. If the user agrees to the migration in Phase 7, ask via `AskUserQuestion` whether they have already done prerequisite steps 1–3 above. If not, list them and **stop** — do not attempt to write tokens or call `vercel login`.
2. If yes, write both workflow files using the chosen `runs-on:` tier.
3. Print a final reminder that they must disconnect Vercel's Git integration **after** verifying the first GHA-driven deploy succeeded, not before — and that the PR should be tested by merging into a throwaway branch first if possible.
4. The workflows above use unpinned `actions/checkout@v2` and `vercel@latest`, which conflict with the supply-chain-auditor skill's recommendations. Note this in the report and recommend running the supply-chain-auditor as a follow-up to pin SHAs and lock the CLI version.

## Important Notes

- Do not modify `generateStaticParams` without explicit user OK — removing it can blow up SEO and performance.
- When converting ISR → SSR + CDN headers, default `s-maxage` to the prior `revalidate` value and `stale-while-revalidate` to 24h, but **ask** before committing.
- Server actions wired to `revalidatePath`/`revalidateTag` are often the load-bearing reason ISR is in place — surface them prominently before suggesting removal.
- Vercel MCP: read-only operations only. Never call `*_delete` or destructive tools.
- No secrets in chat. Vercel auth comes from the MCP / env, not user input.
- If a verdict is uncertain, default to **MAYBE** — never auto-fix on a coin flip.
- This is an audit + assist skill, not a refactor bot. Bias toward fewer, higher-confidence changes.
- Pricing must always come from a live WebFetch of the provider's pricing page. **Do not hardcode or invent $/min, included minutes, or vCPU counts.** If WebFetch fails, emit `pricing unavailable — see <URL>` rather than guessing.
- The build-time projection is explicitly bracketed (best case ≈ vCPU ratio, worst case 1.5×) — never present a single number as authoritative. Always include "rough estimate, actual will vary by build parallelism".
- For `vercel-build` projects, the runner-swap suggestion is an architectural shift (move CI to GitHub Actions). Flag it as such — do not present it as a one-line config edit.
- Cite the source URL and the date the pricing was pulled in every report so the user can verify before acting.
