---
pr: 611
title: "Doc: Add a local hook development guide"
author: linolayani
head_sha: "bf6575f7d15a5902d022415f6f0057bb2f07ef23"
base: main
reviewed_at: "2026-06-08T18:57:49Z"
refresh_log:
  - at: "2026-06-08T18:57:49Z"
    note: "Populated head_sha from live PR; no new commits or activity since review."
verdict: request-changes
gate:
  decision: hold
  gated_at: "2026-06-08T18:59:20Z"
  gated_head_sha: "bf6575f7d15a5902d022415f6f0057bb2f07ef23"
  reviewed_head_sha: "bf6575f7d15a5902d022415f6f0057bb2f07ef23"
---

## Gate

**HOLD** — all three critical findings from the review are unaddressed in the current head. The PR also carries `lifecycle/stale` and `needs-ok-to-test` labels, has no formal reviews, and has not been approved. The content is sound but the guide is non-functional as written.

### Gating list

- **Broken `go run` command** (`getting-started-hook.md` lines ~84-88, `REVIEW.md` critical): `--github-app-private-key-path` and `--github-app-id` are still missing line-continuation backslashes. As written, the shell executes `go run` without those flags and then tries to run the flag strings as separate commands. The primary instruction of the guide does not work. Verified against `FETCH_HEAD` (the current PR head `bf6575f7d`).
- **Typo "sensible" → "sensitive"** (`config/local/hook/README.md:3`, `REVIEW.md` critical + `@ivankatliarchuk`): The sentence still reads "safe place to store sensible credentials". The two post-initial commits touched this file but only changed the heading text and a broken link — the typo remains.
- **Typo "Form" → "From" + incomplete sentence** (`getting-started-hook.md` lines ~74,76, `REVIEW.md` critical + `@ivankatliarchuk`): "Form the root" and the dangling "We're now going to" are both unchanged.

### Process blockers

- `lifecycle/stale` applied 2026-06-06 by triage bot (90 days of inactivity). Author must `/remove-lifecycle stale` to keep the PR alive.
- `needs-ok-to-test`: no org member has run `/ok-to-test`. CI has not run on this PR.
- No reviews submitted; `reviewDecision` is unset. `@krzyzacy` and `@matthyx` are requested but have not engaged.
- NOT APPROVED per approver bot (requires `@matthyx` or root OWNERS approver).

### Area 2 — Independent merge risk

**None.** This PR is documentation-only (`+151 −0` across `.md` files and four config template files). No Go code, no API surface, no CRD schema, no flag changes. Merging cannot break existing deployments.

### What unblocks merge

1. Fix the broken `go run` continuation backslashes.
2. Fix "sensible" → "sensitive" in `config/local/hook/README.md:3`.
3. Fix "Form" → "From" and complete the trailing sentence in `getting-started-hook.md`.
4. Get `/ok-to-test` from an org member so CI can run.
5. Get `lgtm` + `approve` from `@matthyx` (or another root approver).

---

# PR #611: Doc: Add a local hook development guide

**Author:** [@linolayani](https://github.com/linolayani) · First-time contributor  
**Link:** https://github.com/kubernetes-sigs/prow/pull/611  
**Diff:** +151 −0  
**Reviewers requested:** @krzyzacy, @matthyx

---

## Reviewer Perspectives

| Perspective | Role | Verdict |
|---|---|---|
| Code Quality | Senior Go engineer | Request Changes |
| Maintainability | Long-term maintainer | Request Changes |
| Deployment Risk | Platform operator | Low Risk — No Concerns |

---

## Prior Community Review

@ivankatliarchuk has already left a detailed review covering many of the same issues (broken command, typos, HMAC secret, naming confusion). The findings below incorporate and extend that review. Overlapping items are noted.

---

## Required Changes

### CRITICAL — Broken `go run` command
*All 3 reviewers + @ivankatliarchuk*

In `site/content/en/docs/getting-started-hook.md:84-85`, the `go run` command block is missing line-continuation backslashes on the last two flags:

```
                 --github-app-private-key-path=config/local/hook/prow-sandbox.private-key.pem
                 --github-app-id=$GITHUB_APP_ID
```

As written, the shell will execute `go run` without these flags, then try to run `--github-app-id=$GITHUB_APP_ID` as a separate command. The guide's primary instruction is non-functional.

**Suggested comment:**
> The last two flags are missing line-continuation backslashes (`\`). As written, the shell will execute `go run` without `--github-app-private-key-path` and `--github-app-id`, then try to run those as separate commands:
>
> ```suggestion
>                  --hmac-secret-file=config/local/hook/hmac-secret \
>                  --github-app-private-key-path=config/local/hook/prow-sandbox.private-key.pem \
>                  --github-app-id=$GITHUB_APP_ID
> ```

---

### CRITICAL — Typo: "sensible" → "sensitive"
*Code Quality + Maintainability + @ivankatliarchuk*

`config/local/hook/README.md:3` says "safe place to store sensible credentials" — a meaning-altering typo in a security-relevant sentence. Should be "sensitive credentials".

**Suggested comment:**
> ```suggestion
> This folder contains configuration files to run [Hook](/docs/components/core/hook/) locally for plugin development and testing. This is a safe place to store sensitive credentials, as the `config/local` folder is included in the gitignore.
> ```

---

### CRITICAL — Typo: "Form" → "From" + incomplete sentence
*Code Quality + Maintainability + @ivankatliarchuk*

- `getting-started-hook.md:76`: "Form the root" should be "From the root".
- `getting-started-hook.md:74`: "We're now going to" — sentence trails off with no conclusion. Add e.g. "run the two processes."

**Suggested comment:**
> "Form the root" → "From the root"
>
> Also, the sentence "We're now going to" on the line above is incomplete — it trails off. Consider: "We're now going to start Hook and the smee client."

---

## Warnings

### WARNING — `.gitignore` vs tracked files tension
*Maintainability + initial review*

The PR adds `config/local/` to `.gitignore` **and** commits files under `config/local/hook/`. This is a workable but surprising pattern: `.gitignore` only affects *untracked* files. Once committed, git continues tracking these files — modifications show in `git diff`, and `git clean -fdx` will delete them entirely.

The README's claim that this is a "safe place to store sensitive credentials" is **misleading for committed files**. New files created here (like a real private key) *will* be correctly ignored, but edits to committed files (like putting a real secret in `hmac-secret`) will be tracked.

**Suggested alternative:** Place templates in a non-ignored location (e.g. `config/templates/hook/`) and have users copy them into the gitignored `config/local/hook/`. This is the well-understood `.example` / `cp` convention.

**Suggested comment:**
> The `config/local/` directory is gitignored, but these template files are committed into it. `.gitignore` only affects *untracked* files — once committed, git continues tracking changes (they'll show in `git diff`, and `git clean -fdx` will delete them).
>
> This means the README's claim that this is a "safe place to store sensitive credentials" is misleading for any of the committed files. If a user modifies `hmac-secret` with a real value, it *will* show in `git status`.
>
> Consider placing templates in a non-ignored location (e.g. `config/templates/hook/`) with instructions to copy them into `config/local/hook/`. This follows the well-understood `.env.example` → `.env` convention.

---

### WARNING — Hardcoded HMAC secret value
*Code Quality + Deployment Risk context*

`config/local/hook/hmac-secret` contains the literal value `abcde12345`. The Deployment Risk reviewer noted this is the **established convention** across the codebase (`cmd/phony`, integration tests, integration config), so it's consistent — but Code Quality suggested encouraging `openssl rand -hex 20` to avoid the value being cargo-culted into less appropriate contexts.

**Maintainer call:** Non-blocking given the established convention, but worth a brief note in the guide that this is a throwaway dev-only value.

---

### WARNING — `config/local/hook` may be confused with git hooks
*@ivankatliarchuk*

In a git repository, "hook" commonly refers to git hooks (`.git/hooks/`). The directory name `config/local/hook` could confuse contributors. Consider `config/local/prow-hook` or `config/local/hook-dev`.

**Suggested comment:**
> Nit: `config/local/hook` could be confused with git hooks. Consider `config/local/prow-hook` or similar to disambiguate.

---

### WARNING — README links broken outside Hugo

`config/local/hook/README.md` uses relative links (`/docs/components/core/hook/`, `/docs/getting-started-hook`) that only resolve when rendered by Hugo. They are broken when reading the README on GitHub or in an editor. Consider using full GitHub URLs or noting these resolve on the published site.

---

## Nits & Polish

### INFO — Multiple grammar/typo issues in `getting-started-hook.md`
*Code Quality + @ivankatliarchuk*

| Location | Current | Fix |
|---|---|---|
| Set up webhook, intro | "smee… to forwards" | "to forward" |
| Test the setup | "This plugin listen for" | "This plugin listens for" |
| Test the setup | "listen for /woof comment" | "listens for `/woof` comments" |
| Understanding, step 1 | "eg." | "e.g." |
| Understanding, step 3 | "smee-client receive" | "receives" |
| Understanding, step 4 | "parse the event" | "parses" |
| Understanding, step 5 | "adding a comment the issue" | "adding a comment to the issue" |
| Understanding, step 5 | "eg." | "e.g." |

---

### INFO — Hugo heading level

`# Prerequisites` uses h1 while all subsequent sections use `##` (h2). Hugo content with frontmatter `title:` should start body headings at `##`.

---

### INFO — Missing language annotation on code block

The smee command block (step 2 under "Run hook and smee") uses a bare ` ``` ` fence. Add ` ```sh ` for syntax highlighting consistency.

---

### INFO — Missing trailing newline in `hmac-secret`

`config/local/hook/hmac-secret` lacks a final newline. Some tools behave unexpectedly with files missing one, and Hook's HMAC reading code may or may not trim whitespace.

---

### INFO — `.gitignore` comment wording

"Make sure to not commit sensitive credentials" → "Do not commit sensitive credentials."

---

## What's Good

- The smee-based approach is practical and well-chosen for local development.
- Using `go run ./cmd/hook` avoids build-step dependencies — keeps the guide simple.
- Flag names correctly match `cmd/hook/main.go` and `pkg/flagutil/github.go`.
- The file table in the README is helpful for orientation.
- The "Understanding the workflow" section builds mental models for newcomers.
- The `config/local/` pattern is extensible to other components beyond Hook.
- The `.pem` placeholder correctly uses only comments (no dummy key material).
- The HMAC placeholder value is consistent with the established codebase convention.
- Linking to the full documented config reference helps discoverability.
- Overall concept meaningfully lowers the barrier to entry for new contributors.

---

## Verdict

**Request Changes**

Broken shell command makes the guide non-functional. Three text fixes also required. Once addressed, deployment risk is zero and maintenance burden is low — ready to merge.

---

## Draft Overall Comment

> Thanks for putting this guide together @linolayani — it fills a real gap in the onboarding experience. I'm requesting changes for one critical fix and a few small text corrections.
>
> The `go run` command on lines 84-85 is missing line continuation backslashes, which means anyone following the guide will hit a shell error. Please also fix the typos ("sensible" → "sensitive", "Form" → "From") and complete the truncated sentence ("We're now going to").
>
> Once those are addressed, this is good to merge. The gitignore/template layout and heading consistency points are suggestions you can take or leave.

---

## Review Checklist

- [ ] Post overall comment
- [ ] Post inline comment: broken go run command (lines 84-85)
- [ ] Post inline comment: "sensible" → "sensitive" typo (README.md:3)
- [ ] Post inline comment: "Form" → "From" + incomplete sentence (lines 74, 76)
- [ ] Post inline comment: gitignore/tracked files tension (non-blocking suggestion)
- [ ] Optionally post: README broken links, naming confusion, heading levels, trailing newline
- [ ] Submit review as "Request Changes"
