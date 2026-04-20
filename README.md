# Skills

A collection of reusable agent skills compatible with the [skills](https://www.npmjs.com/package/skills) CLI.

## Install

```bash
npx skills@1.4.9 add erezrokah/skills
```

## Available Skills

### supply-chain-auditor

Audit a repository for supply chain security risks. Detects ecosystem type, runs category-specific checks, and produces an actionable report with severity ratings.

### you-dont-need-isr

Audit a Next.js project's Incremental Static Regeneration usage. Classifies each occurrence as necessary or overkill, projects build cost on a stronger runner (live Ubicloud + Vercel pricing), and walks through replacing overkill cases with simpler alternatives (SSG + webhook, SSR + CDN headers, client fetch).
