---
pr: kubernetes-sigs/prow#844
title: "fix(site): upgrade postcss-cli to 11 to patch picomatch CVE-2026-33672"
head_sha: 701be5a93463a0175a9ff678a39015ab057a24ab
base: main
reviewed_at: 2026-08-12T10:20:49Z
verdict: approve
---

## What this PR does
- Bumps `postcss-cli` `9.1.0 -> 11.0.1` in `site/package.json`, regenerating `site/package-lock.json`.
- This replaces vulnerable `picomatch 2.3.1` with patched `2.3.2` (existing chokidar subtree) and introduces a new `tinyglobby` subtree pulling in patched `picomatch 4.0.5`.
- Refreshes `autoprefixer` `10.4.13 -> 10.5.4` (within existing `^10.4.2` range; lockfile refresh only, not a manifest change).
- Pins `NODE_VERSION = "22"` in `netlify.toml`, required because postcss-cli 11 needs Node >=18.

## Findings
(none)

## Checked
- Classification: dep-only PR. `git diff --stat` against `upstream/main` touches only `netlify.toml`, `site/package.json`, `site/package-lock.json` — no project source.
- Freshness of all bumped/new versions relative to 2026-08-12: postcss-cli 11.0.1 (2025-03-12, >1yr), picomatch 2.3.2 (2026-03-23, ~4.5mo), picomatch 4.0.5 (2026-07-02, ~6wk), tinyglobby 0.2.17 (2026-05-30, ~2.5mo), autoprefixer 10.5.4 (2026-07-16, ~27d). All past the freshness threshold; none are pseudo-versions.
- CVE resolution verified against GitHub advisories: GHSA-3v7f-55p6-f55p (medium, POSIX method injection) and GHSA-c2c7-rcm5-vvqj (high, extglob ReDoS) both list picomatch `<2.3.2` and `<4.0.4` as vulnerable; lockfile diff confirms only `2.3.2` and `4.0.5` remain, no `2.3.1`.
- postcss-cli changelog 9.1.0->11.0.1 reviewed (10.0.0: drop Node12, ESM config; 10.1.0: non-TTY watch; 11.0.0: require Node>=18, postcss-load-config@5; 11.0.1: dep minimization only). No CSS-processing behavior changes expected.
- Node >=18 requirement from postcss-cli 11 is correctly covered by the new `NODE_VERSION = "22"` pin in `netlify.toml`.
- Usage/exposure: `postcss-cli`/`autoprefixer` are devDependencies used only to build the static docs site (Hugo + PostCSS via Netlify); not vendored into any Prow Go binary, no runtime/production exposure.
- PR description states `npm audit` reports 0 vulnerabilities post-change and `npm ls picomatch` shows only patched versions.

## Open questions
- The PR's own test plan checkbox for the Netlify deploy preview building successfully is unchecked — worth confirming the preview is green before merge, since it's the only end-to-end validation of the Hugo/PostCSS/autoprefixer pipeline with the new toolchain.
