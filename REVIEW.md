---
pr: kubernetes-sigs/prow#740
title: "docs: add operator-facing GitHub API access documentation"
head_sha: 968c8c5f3e4472ee321477d3be45c0d16945b884
base: main
reviewed_at: 2026-07-27T00:26:51Z
verdict: request-changes
---

## What this PR does

- Adds `site/content/en/docs/github-api-access.md`: central operator doc covering GitHub auth (PAT vs GitHub App), rate-limit management (client-side throttling, ghproxy), and endpoint configuration (REST/GraphQL/Search).
- Adds short "GitHub API Access" sections to hook, crier, deck, tide, branchprotector, status-reconciler docs, each linking back to the central page.
- Docs-only change, +115/-3 across 7 files. PR is already merged (has `lgtm`/`approved` labels); findings below are for a fast follow-up.
- Confirmed via a 3-perspective maintainer review (code quality, maintainability, deployment risk) — all three independently converged on the same core errors.

## Findings

### [blocking] Self-contradictory rate-limit comparison
- where: `site/content/en/docs/github-api-access.md` (GitHub App Authentication section, "Higher rate limits" bullet)
- concern: States "GitHub Apps receive higher API rate limits (5,000 requests/hour per installation vs 5,000/hour for personal tokens)" — same number on both sides of a claimed contrast. GitHub App installation limits actually scale with repos/users covered (baseline higher than 5,000, commonly 12,500-15,000+). Flagged independently by all three maintainer-review perspectives.
- excerpt: |
    - **Higher rate limits:** GitHub Apps receive higher API rate limits (5,000 requests/hour per installation vs 5,000/hour for personal tokens)

### [blocking] Fabricated flag `--github-search-endpoint`
- where: `site/content/en/docs/github-api-access.md` (Configuring Components to Use ghproxy section, endpoint flag list and example config block)
- concern: `--github-search-endpoint` does not exist anywhere in the codebase (checked `pkg/flagutil/github.go` and grepped all `.go` files). Search requests (`FindIssues`/`FindIssuesWithOrg`) use the same REST base configured by `--github-endpoint`. Deployment-risk review flags this as the one item with real blast radius: if an operator copy-pastes the example into a component's args, Go's flag package rejects the unrecognized flag and the rollout fails to start.
- excerpt: |
    - `--github-search-endpoint`: Search API endpoint (often the same as REST)

### [should-fix] Inaccurate, non-per-component throttling flag defaults
- where: `site/content/en/docs/github-api-access.md` (Client-side Throttling section)
- concern: Claims `--github-hourly-tokens` defaults to "matches GitHub's limit" and `--github-allowed-burst` defaults to "100" as if universal. Per `pkg/flagutil/github.go` (`ThrottlerDefaults`), both actually default to `0` (disabled) unless a component opts in explicitly. `hook`, `deck`, `crier` (three of the six components this PR documents) never opt in — throttling is off by default for them. `branchprotector`/`status-reconciler`/`peribolos` use 300/100, `invitations-accepter` uses 300/300, `needs-rebase` uses 360/360. No single default matches what's stated.
- excerpt: |
    - `--github-hourly-tokens`: Maximum GitHub tokens to consume per hour (default: matches GitHub's limit)
    - `--github-allowed-burst`: Maximum burst of tokens that can be consumed at once (default: 100)

### [should-fix] Tide doesn't expose the documented throttling flags at all
- where: `site/content/en/docs/components/core/tide/_index.md` (GitHub API Access section) and `github-api-access.md` (Client-side Throttling section)
- concern: `cmd/tide/main.go` calls `AddCustomizedFlags(fs, prowflagutil.DisableThrottlerOptions())`, so `--github-hourly-tokens`/`--github-allowed-burst` aren't registered on tide at all. Tide instead has its own `--sync-hourly-tokens` (default 800) and `--status-hourly-tokens` (default 400). The tide doc section links to the central page's generic throttling guidance with no callout of this divergence — an operator following it could try a flag tide rejects, or miss the flags that actually matter.
- excerpt: |
    Tide is a heavy consumer of the GitHub API. ... It requires GitHub authentication credentials and should be configured with ghproxy to manage rate limits. See [Managing GitHub API Access](/docs/github-api-access/) ...

### [nit] Overgeneralized "flags can be specified multiple times" claim
- where: `site/content/en/docs/github-api-access.md` (Configuring Components to Use ghproxy section)
- concern: True for `--github-endpoint` (repeatable `Strings` flag) but `--github-graphql-endpoint` is a single string flag with no fallback list — the blanket claim across all endpoint flags overstates GraphQL's behavior.
- excerpt: |
    When multiple endpoints are provided for the same API type, components try them in order and fall back to the next if one is unavailable.

### [nit] Empty frontmatter description
- where: `site/content/en/docs/github-api-access.md:1-5`
- concern: `description: >` has no content after it — most other docs pages populate this for site listings/SEO.
- excerpt: |
    ---
    title: "Managing GitHub API Access"
    weight: 104
    description: >

    ---

### [nit] Lost concrete detail in crier.md
- where: `site/content/en/docs/components/core/crier.md`
- concern: Removed lines specified the conventional token mount path (`--github-token-path` defaulting to `/etc/github/oauth`), still used in example deployment YAMLs (e.g. `test/integration/config/prow/cluster/crier_deployment.yaml`). The generic replacement text doesn't preserve this detail.
- excerpt: |
    -You also need to mount a github oauth token by specifying `--github-token-path` flag, which defaults to `/etc/github/oauth`.
    -
    -If you have a [ghproxy](/docs/ghproxy/) deployed, also remember to point `--github-endpoint` to your ghproxy to avoid token throttle.

### [nit] No reciprocal link from ghproxy docs
- where: `site/content/en/docs/ghproxy/_index.md`
- concern: The new page links to ghproxy, but ghproxy's own doc doesn't link forward to the new central page — operators landing on ghproxy first won't discover the broader operational overview.

### [question] Search API rate-limit claim unverifiable from this repo
- where: `site/content/en/docs/github-api-access.md` (Managing API Rate Limits section)
- concern: "Search API: 30 requests/minute (shared across v3 and v4)" is a claim about GitHub's own API, not something prow implements/asserts in code — no way to confirm or refute it from this codebase. Worth double-checking against current GitHub docs.
- excerpt: |
    - **Search API:** 30 requests/minute (shared across v3 and v4)

## Checked

- "Cached responses don't count against the token budget" claim — confirmed correct. `pkg/github/client.go:447-461` explicitly refunds the throttling token when `ghcache.CacheModeIsFree(cacheMode)` is true, matching existing wording in `site/content/en/docs/ghproxy/_index.md`.
- ETag/conditional-request handling claim — confirmed against `pkg/ghcache/ghcache.go`.
- Flag names `--github-token-path`, `--github-app-id`, `--github-app-private-key-path`, `--github-hourly-tokens`, `--github-allowed-burst`, `--github-endpoint`, `--github-graphql-endpoint` all exist verbatim in `pkg/flagutil/github.go:133-141`.
- Component-doc additions (hook, crier, deck, tide, branchprotector, status-reconciler) are consistent in structure/tone, correctly link back to the central page rather than duplicating detail, and match an existing established pattern (same style already used for ghproxy links). `weight: 104` slots cleanly into nav ordering with no collision.
- No doc-linter or doc-gen mechanism exists tying these prose numbers to `pkg/flagutil/github.go` — flagged as a maintainability risk (future drift) rather than a current defect beyond what's listed above.

## Open questions

- Where does the "5,000 requests/hour per installation" figure for GitHub Apps come from — was this meant to be a different (higher) number?
- Is `--github-search-endpoint` intended for a future flag that doesn't exist yet, or was it just an assumption that should be removed?
- Can the throttling defaults section be reworded to reflect that flags default to disabled (0) and defaults are set per-component, rather than implying a single universal default?
- Should the tide doc section explicitly mention `--sync-hourly-tokens`/`--status-hourly-tokens` instead of pointing at the generic throttling flags?
