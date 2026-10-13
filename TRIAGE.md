---
issue: 658
title: "Deck: Surface Debugging Hints"
repo: kubernetes-sigs/prow
author: BenTheElder
filed: "2026-03-16"
state: open
component: Deck / Spyglass
category: feature
verdict: legitimate
effort: level-2
triaged_at: "2026-06-15T10:42:08Z"
refresh_log:
  - at: "2026-04-14"
    summary: "Initial triage"
  - at: "2026-06-15T10:42:08Z"
    summary: "lifecycle/stale added by bot (90d inactivity), immediately removed by pohly — no substantive change"
labels_recommended:
  - area/spyglass
  - kind/feature
  - help-wanted
---

# Triage: Issue #658 — Deck: Surface Debugging Hints

**Author:** BenTheElder (Benjamin Elder) — SIG Testing
**Filed:** 2026-03-16
**State:** OPEN, no labels
**Component:** Deck / Spyglass
**Category:** Feature Request

## Overview

Request to render a `$ARTIFACTS/DEBUGGING.md` file as a semi-collapsed viewer near the top of Spyglass test results pages. Goal: help contributors (especially new ones) discover useful debug artifacts beyond raw build logs.

The "artifacts" link is one of many on the page and doesn't stand out. Useful debug info (apiserver logs, metrics) is buried and requires tribal knowledge to find.

**Since previous triage (2026-04-14):**

- 2026-06-14: `k8s-triage-robot` applied `lifecycle/stale` (90 days of inactivity trigger).
- 2026-06-15: `pohly` ran `/remove-lifecycle stale`, keeping the issue active.
- No new substantive comments, no linked PRs, no state change.

### Discussion Highlights

- **pohly**: Worried about burden on job maintainers — they already don't do optional improvements. Something automatic for all jobs would be more realistic.
- **BenTheElder**: Imagines tools like kubetest2 deployers emitting DEBUGGING.md alongside artifacts. Automatic per-deployer, not per-job.
- **petr-muller**: OpenShift already uses the existing `html` lens for this — surfacing HTML files dropped into collected artifacts.

## Spyglass Lens Architecture

Spyglass uses a **plugin-based lens system**. Each lens is a Go package implementing `Header()`, `Body()`, `Callback()`. Lenses register via `init()` and are imported as side-effect imports in `cmd/deck/main.go`. Config maps artifact filename regex patterns to lenses.

### Key Code Paths

| What | Where |
|------|-------|
| Lens interface | `pkg/spyglass/lenses/lenses.go:54-77` |
| Lens registration | `pkg/spyglass/lenses/lenses.go:85-98` |
| Artifact matching | `cmd/deck/main.go:1031-1057` |
| Lens ordering (priority) | `pkg/spyglass/spyglass.go:106-142` |
| Lens HTTP handler | `pkg/spyglass/lenses/common/common.go:102-155` |
| Config structs | `pkg/config/config.go:1116-1159` |
| Frontend (iframes) | `cmd/deck/static/spyglass/spyglass.ts` |

### Existing Lenses

| Lens | Priority | What it does |
|------|----------|--------------|
| `buildlog` | 10 | Plain text log rendering with error highlighting |
| `html` | 3 | Renders HTML artifacts in sandboxed iframe |
| `junit` | — | JUnit XML test results |
| `metadata` | 0 | Structured JSON (started.json, finished.json) |
| `links`, `podinfo`, `coverage`, `restcoverage` | — | Specialized viewers |

`RequiredFiles` — all regexes must match ≥1 artifact (AND). `OptionalFiles` — any regex can match (OR).

**Markdown infrastructure:** Only `pkg/markdown/code_block.go` (strips code blocks). No markdown-to-HTML renderer exists in the codebase.

## Root Cause: Feature Gap

- Spyglass treats all artifacts equally — no mechanism to "promote" certain artifacts
- No markdown rendering lens exists
- Lens system requires explicit per-deployment config — jobs can't self-declare their lenses

## Proposed Solutions

### Approach 2: Generic Markdown Lens (RECOMMENDED)

General-purpose markdown rendering lens (`pkg/spyglass/lenses/markdown/`) configurable to match any `.md` artifact. DEBUGGING.md becomes one configuration.

**Pros:**
- Versatile — any .md artifact
- Markdown simpler for maintainers than HTML
- Follows established lens patterns
- Fully backwards compatible

**Cons:**
- Adds goldmark dependency
- Less opinionated UX

### Approach 1: Dedicated Debugging Lens

Purpose-built lens matching `DEBUGGING.md` specifically.

**Pros:** Matches issue request exactly; built-in collapsible display
**Cons:** Less versatile (single use case); same dependency cost

### Approach 3: Leverage Existing HTML Lens

Jobs emit `DEBUGGING.html` instead. Zero Prow changes. OpenShift already does this.

**Pros:** Works today, proven in production
**Cons:** HTML harder for maintainers to produce; low priority (3), HideTitle=true — not prominent; doesn't address discoverability

## Effort Assessment

**LEVEL 2 — MODERATE** | help-wanted

Well-patterned task with clear reference implementations, but spans multiple files and requires markdown library + HTML sanitization.

| Factor | Assessment | Level |
|--------|-----------|-------|
| Scope | ~5-8 files, ~200-400 LOC | 2-3 |
| Complexity | Simple interface, main challenge is markdown lib + sanitization | 2 |
| Required Expertise | Go templates, lens API, HTML sanitization (learnable) | 2 |
| Clarity | Core feature clear; open questions on link handling, markdown features | 2-3 |
| Testing | Follow existing lens test patterns | 2 |
| Backwards Compat | Fully compatible — purely additive, opt-in | 1 |
| Arch Alignment | Perfect fit — lens system designed for this | 1 |
| External Deps | goldmark (mature, MIT) | 1-2 |

## Recommended Labels

- `area/spyglass`
- `kind/feature`
- `help-wanted`

No `good-first-issue` — requires understanding multiple components and HTML sanitization.
No priority label — enhancement, not urgent.
No retitle — current title is clear.

## Proposed Comment

Spyglass already has the infrastructure to support this well. The lens plugin system (`pkg/spyglass/lenses/`) allows creating artifact viewers that match files by regex pattern — a new lens matching `DEBUGGING.md` (or any `.md`) would automatically render when that artifact is present. Existing lenses like `html` (`pkg/spyglass/lenses/html/`) and `metadata` provide clear implementation patterns to follow: implement `Header()`, `Body()`, `Callback()`, register via `init()`, and configure artifact matching in the Spyglass config.

A general-purpose **markdown lens** (rather than debugging-specific) would be the most versatile approach — it could render any `.md` artifact, with the DEBUGGING.md use case being one configuration. This would need a Go markdown library (e.g., goldmark) for server-side rendering with HTML sanitization. Lens priority controls placement, so a DEBUGGING.md config could be set to appear prominently near the top of the Spyglass view. As @petr-muller noted, OpenShift already uses the existing `html` lens to surface similar content — that works today as a workaround if jobs emit HTML instead of markdown, though markdown is simpler for job maintainers to produce (addressing @pohly's concern about burden).

```
/area spyglass
/kind feature
/help-wanted
```
