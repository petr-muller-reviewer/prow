---
pr: kubernetes-sigs/prow#1019
title: "chore(deps): bump source-map-js from 1.2.1 to 1.2.2 in /site"
head_sha: 6f9515bb077a3d19c22bbed816cea7923a442731
base: main
reviewed_at: 2026-10-22T00:18:11Z
verdict: approve
gate:
  decision: merge
  gated_at: 2026-10-22T00:19:38Z
  gated_head_sha: 6f9515bb077a3d19c22bbed816cea7923a442731
  reviewed_head_sha: 6f9515bb077a3d19c22bbed816cea7923a442731
---

# Review

## Gate

**Verdict:** merge.

The PR is open and unchanged since the reviewed head `6f9515bb077a3d19c22bbed816cea7923a442731`. The prior review has no blocking or should-fix findings, and the PR has no GitHub reviews or substantive reviewer comments. The complete diff remains a single indirect npm development-dependency lockfile update, so it introduces no API, configuration, or runtime behavioral change for existing Prow deployments.

**Findings disposition:** all important findings are addressed; there were none.

**Gating list:** none.

**Merge risk:** no notable merge risk. No repository-specific compatibility skill applies to an npm lockfile-only change.

## Verdict

Approve. This is a clean, dep-only lockfile update that moves an indirect npm development dependency to the release that fixes a published high-impact availability vulnerability. Exposure in Prow is limited to the static site's PostCSS build pipeline.

## What this PR does

- Updates `source-map-js` from `1.2.1` to `1.2.2` in `site/package-lock.json`.
- Changes only the version, resolved tarball URL, and integrity digest for that package.
- Leaves the site dependency manifests and project source unchanged.

## Findings

None.

## Checked

### Classification

Dep-only. The PR changes only `site/package-lock.json`; no project code, manifests, configuration, docs, or tests change.

### source-map-js 1.2.1 → 1.2.2

- **Freshness:** `1.2.2` was released on GitHub on 2026-09-30 and published to npm at 2026-09-30T14:08:09Z. It is 22 days old as of this review. That is sufficient soak time, and this release is itself the security fix.
- **Usage:** Indirect development dependency. Prow does not import `source-map-js` directly and has no source references to it. The sole dependency path is `site` → `postcss@8.5.26` → `source-map-js@1.2.2`; `postcss` is declared as a development dependency in `site/package.json:4`, and the lockfile marks `source-map-js` as `dev: true` at `site/package-lock.json:708`. The package provides source-map parsing/generation for PostCSS.
- **Vulnerabilities:** OSV reports `GHSA-68fv-2mgg-jv7q` / `CVE-2026-93749` for `1.2.1`, fixed in `1.2.2`. The issue is event-loop denial of service from malicious indexed source maps with extremely large section offsets, with high availability impact. OSV reports no advisory for `1.2.2`. `govulncheck: not run (npm dependency; OSV used instead).`
- **Changelog and exposure:** Upstream made four commits. The substantive changes are a fix for browser execution under restrictive CSP `script-src`, plus the indexed-source-map DoS fix and tests. Neither runtime server code nor the shipped website directly invokes this library; Prow only reaches it through PostCSS while building the static site. The project does not process untrusted external source maps on that path, so exposure is light and build-time only. The security fix nevertheless makes the update worthwhile.
- **Take:** Safe to bump now.

### Validation

- `site` lockfile install dry run: `npm install --package-lock-only --ignore-scripts --dry-run` succeeds and reports the tree as up to date.
- PR checks: `deploy/netlify` and `pull-prow-verify-lint` pass; EasyCLA passes. Tide remains pending because the PR lacks approval/LGTM labels.

## Open questions

None.

## Dependency followups

No opportunities identified.

Examined the sole changed dependency, `source-map-js 1.2.1 → 1.2.2`. Its release notes contain a browser CSP crash fix and the indexed-source-map DoS fix, but Prow does not import or call `source-map-js` directly. The package is reached only indirectly through the site's development-time PostCSS pipeline, so no deprecation, API upgrade, new feature, simplification, or default change applies to project-owned call sites.
