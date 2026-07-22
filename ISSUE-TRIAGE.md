# Triage for Issue #589

**Status**: In Progress
**Created**: 2026-07-22

## Issue Information

- **Issue Number**: #589
- **Issue URL**: https://github.com/kubernetes-sigs/prow/issues/589

## Findings

### Initial Validation

**Assessment**: LEGITIMATE

**Analysis**

This issue proposes a refactoring based on an existing TODO comment in the codebase. The TODO exists at `pkg/git/v2/client_factory.go:107` and identifies a code quality issue: the current implementation uses two boolean pointer fields (`UseInsecureHTTP` and `UseSSH`) to represent three mutually exclusive schemes (HTTPS, HTTP, SSH). Verified against current `main` (this worktree) — the TODO comment and struct fields are unchanged since the issue was filed.

**Issue Category**: Enhancement/Refactoring

**Repository Scope Check**:
- Component mentioned: `pkg/git/v2` client factory
- Exists in this repo: Yes
- Relevant code paths:
  - `pkg/git/v2/client_factory.go` (struct at lines ~102-128, TODO at line 107)

**Information Completeness**:
- Sufficient detail provided: Yes
- Missing information: None critical
- The issue quotes the exact TODO comment and proposes a concrete solution (a `SchemeType` enum)
- Author (`tsj-30`, a repo member) self-assigned via `/assign` on 2026-01-12 and confirmed intent to draft a PR

**Activity Since Filing**:
- 2026-01-12: Author self-assigned (`/assign`); maintainer `matthyx` said no specific requirements, deferred to iteration; author agreed to draft a PR
- 2026-01-16: `petr-muller` (maintainer) also deferred, gave a green light to "take a shot at it"
- 2026-04-16: `k8s-triage-robot` applied `lifecycle/stale` after 90 days of inactivity
- 2026-04-17: Author removed `lifecycle/stale`
- 2026-07-16: `k8s-triage-robot` re-applied `lifecycle/stale` after another ~90 days of inactivity — **currently stale, no PR has materialized** despite two maintainer green-lights and an assignment

**Current Implementation Analysis**:
The current design uses two optional boolean pointers to encode three states:
- Default/both-nil/both-false → HTTPS
- `UseInsecureHTTP = true` → HTTP (overrides UseSSH per comment)
- `UseSSH = true` → SSH

This creates ambiguity (what if both are true?) and makes the API less clear than an explicit enum would be.

### Recommendation

**Suggested Action**: Keep open and continue triage

This is a legitimate, maintainer-approved refactoring request addressing a documented TODO. The issue is well-written, was explicitly greenlit by two maintainers, and the author self-assigned — but six months later no PR has appeared and the issue is currently `lifecycle/stale`. This is worth flagging in the augmentation/comment: either nudge the assignee for status, or open the assignment back up if they're no longer working on it.

Next steps: Proceed with research phase to identify all code locations that would need updating and assess implementation effort.

### Code Research

**Primary Components**:
- `ClientFactoryOpts` struct (pkg/git/v2/client_factory.go:102-128) — configuration options for the git client factory; holds `UseInsecureHTTP *bool` (108) and `UseSSH *bool` (110)
- `NewClientFactory` (pkg/git/v2/client_factory.go:292-341) — constructs the factory, selects a `RemoteResolverFactory` based on scheme options
- `httpResolverFactory` (pkg/git/v2/remote.go:87-152) — HTTP/HTTPS remote URL resolver; `http bool` field toggles scheme lazily at resolve time
- `sshRemoteResolverFactory` (pkg/git/v2/remote.go:57-85) — SSH resolver; has no scheme concept at all (always `git@host:org/repo.git`)
- `gerritResolverFactory` (pkg/git/v2/remote.go:176-192) — Gerrit resolver, selected via non-empty `CookieFilePath`, not a boolean
- `WithInsecureHTTP` / `WithSSH` (client_factory.go:210-222) — functional-option constructors; **note: nothing in the repo actually calls these constructors** — every caller sets the struct fields directly instead

**Architecture Overview**:
Strategy pattern: `NewClientFactory` picks one of three `RemoteResolverFactory` implementations (ssh, http/https, gerrit) based on `ClientFactoryOpts` fields, then hands the factory to the client for later URL resolution.

**Key Code Paths**:
1. Struct definition: client_factory.go:102-128
2. Option merging: `Apply` method, client_factory.go:157-189 (lines 162-166 copy the two scheme booleans)
3. Scheme decision logic, client_factory.go:314-329:
   ```go
   if o.UseSSH != nil && *o.UseSSH {
       remote = &sshRemoteResolverFactory{host: o.Host, username: o.Username}
   } else if o.CookieFilePath != "" {
       remote = &gerritResolverFactory{}
   } else {
       remote = &httpResolverFactory{host: o.Host, http: o.UseInsecureHTTP != nil && *o.UseInsecureHTTP, username: o.Username, token: o.Token}
   }
   ```
   Priority is SSH > Gerrit > HTTP(S). **Confirmed**: if both `UseSSH` and `UseInsecureHTTP` are true, SSH silently wins and `UseInsecureHTTP` is never even read (it's only referenced inside the unreachable `else` branch) — the "overrides" behavior mentioned in the field's doc comment is real but entirely unenforced/untested.
4. URL generation: `httpResolverFactory.resolve`, remote.go:132-137 — `scheme := "https"; if f.http { scheme = "http" }`

**Data Flow**:
Caller builds `ClientFactoryOpts` (directly or via `Apply`) → `NewClientFactory` applies option funcs → branches on scheme fields to pick a resolver factory → resolver's `resolve()` closure is invoked lazily on each `CentralRemote`/`PublishRemote` call to build the actual git remote URL.

### Related Code

**Repo-wide usage** (grep across the whole repository, not just pkg/git/v2):
- `pkg/git/v2/client_factory.go` — definition and decision logic (as above)
- `test/integration/test/moonraker_test.go:148,382` — the **only** places that set `UseInsecureHTTP` (both `= &trueVal`, set via an inline option closure, not via `WithInsecureHTTP`). Neither call site sets `UseSSH`.
- `pkg/flagutil/github.go:339-344,360` — the one real **production** caller that builds a `ClientFactoryOpts` and passes `opts.Apply` into `NewClientFactory`. Notably, this production path sets `Host`, `CookieFilePath`, `Persist`, `Censor`, `Username`, `Token` — but **never sets `UseSSH` or `UseInsecureHTTP`**. In other words, today's only production code path can never select SSH or insecure-HTTP at all; that capability is exercised exclusively by the integration test harness.

**Similar Functionality**: Gerrit selection already avoids the boolean-flag pattern by using a non-empty `string` (`CookieFilePath`) as its trigger, rather than a `*bool`.

**Existing enum precedent to mirror**: none found. Searched `pkg/git/v2`, `pkg/git` (v1), `pkg/flagutil`, `pkg/gerrit`, and repo-wide for iota-based scheme/protocol/mode enums — nothing comparable exists nearby. This would be a novel pattern for the package, not a case of applying an established local convention.

### Test Coverage

**Existing Tests** (pkg/git/v2/remote_test.go):
- `TestSSHRemoteResolverFactory` (51-112) — SSH URL generation and username-getter error handling. No scheme testing applicable (SSH has none).
- `TestHTTPResolverFactory_NoAuth` (114-133) — constructs `httpResolverFactory{host: ...}` leaving `http` at its zero value (`false`); only exercises the `"https"` branch.
- `TestHTTPResolverFactory` (135-211) — auth/username/token cycling, again always with `http: false`.

**Test Gaps**:
- No test ever sets `http: true` on `httpResolverFactory` — the `"http"` scheme branch (remote.go:135) is completely uncovered.
- `pkg/git/v2/client_factory_test.go` **does not exist** — there is no unit test at all for `NewClientFactory`'s scheme-selection branch (client_factory.go:314-329): nothing verifies `WithSSH`/direct-field-set produces an `sshRemoteResolverFactory`, that insecure-HTTP produces `httpResolverFactory{http:true}`, or that `CookieFilePath` produces `gerritResolverFactory`.
- The `UseSSH`+`UseInsecureHTTP`-both-true precedence behavior is entirely untested, as is `UseSSH`+`CookieFilePath` both set.
- `ClientFactoryOpts.Apply` itself has no dedicated unit test (contrast with e.g. `pkg/config/branch_protection.go`'s `Apply`, covered by `branch_protection_test.go:314`).

### Root Cause Analysis

**Primary Cause**: Classic tri-state-boolean anti-pattern — two independent `*bool` fields are used to represent three mutually exclusive states, instead of one explicit enum.

**Contributing Factors**:
1. Undocumented-in-code, unenforced precedence: the doc comment on `UseInsecureHTTP` says it's "overridden" by SSH, but nothing validates or rejects the both-true case — confirmed by code reading, and confirmed untested.
2. Implicit default (HTTPS) requires reading the struct/branch logic to discover.
3. `httpResolverFactory` duplicates scheme state in its own `http bool` field, derived from `UseInsecureHTTP` at construction time — an extra indirection that an enum would remove.
4. No test exists to catch a regression in the precedence/priority order if it's ever accidentally changed.

### Proposed Solutions

#### Approach 1: Enum-Based Scheme Type (single non-pointer enum)

**Description**: Replace `UseInsecureHTTP *bool` and `UseSSH *bool` with a single `Scheme SchemeType` field; `SchemeType` is an iota-based enum with `SchemeHTTPS` as the zero value, plus `SchemeHTTP` and `SchemeSSH`.

**Pros**:
- Matches the TODO's own suggested fix verbatim
- Eliminates the impossible/unenforced both-true state entirely — compiler-level exhaustiveness via a single switch
- Removes the duplicate `http bool` field currently carried inside `httpResolverFactory`
- Given the "Related Code" finding above, the **only** production caller (`pkg/flagutil/github.go`) never sets these fields at all, so its behavior is fully preserved with zero changes required there

**Cons**:
- Breaking API change to the public `ClientFactoryOpts` struct — removes two fields, changes `WithInsecureHTTP`/`WithSSH` signatures or removes them
- The one real caller that *does* set these fields today (`test/integration/test/moonraker_test.go`, 2 call sites) must be updated

**Affected Components**:
- `ClientFactoryOpts`: replace 2 fields with 1
- `NewClientFactory`: collapse 3-way if/else-if/else into a switch on `Scheme`
- `WithInsecureHTTP`/`WithSSH`: replace with `WithScheme(SchemeType)`, or delete (nothing in-repo calls them today)
- `Apply`: copy the new field instead of the two old ones
- `httpResolverFactory`: can keep its internal `http bool` (set from `scheme == SchemeHTTP`) or be simplified further
- `test/integration/test/moonraker_test.go`: 2 call sites to update
- New tests: `client_factory_test.go` (doesn't exist yet) for scheme selection; extend `remote_test.go` for the `http: true` gap

**Complexity**: Medium — mechanical refactor, but touches public API surface

**Backwards Compatibility**: Breaking, but blast radius is minimal per the "Related Code" findings — exactly one in-repo caller (an integration test) uses the fields being removed; the sole production caller is unaffected.

#### Approach 2: Deprecation-Based Migration

**Description**: Add a new `Scheme SchemeType` field alongside the existing booleans; keep old fields working via a compatibility shim in `Apply`/`NewClientFactory` during a transition period, with deprecation comments.

**Pros**: No immediate break; gradual migration path.

**Cons**: Given there is exactly one in-repo caller of the fields being deprecated (an integration test in this same repo, not an external consumer), a deprecation period adds real implementation and review cost for no actual compatibility benefit — there's no external consumer to protect.

**Complexity**: High relative to the benefit it provides here.

**Backwards Compatibility**: Compatible, but unnecessarily so given the caller inventory above.

#### Approach 3: Enum with Pointer (`Scheme *SchemeType`)

**Description**: Preserve "unset" semantics with a pointer-to-enum instead of a value enum.

**Pros**: Slightly less disruptive than Approach 1; still eliminates the impossible state.

**Cons**: Pointer-to-enum is non-idiomatic Go; still a breaking change; adds nil-check complexity for no real benefit since a zero-value enum (`SchemeHTTPS = 0`) already gives a clean "unset = default" semantic for free.

**Complexity**: Medium

**Backwards Compatibility**: Breaking, same blast radius as Approach 1, but a less clean end state.

#### Recommendation

**Preferred Approach**: Approach 1 (plain value enum).

**Rationale**:
1. Matches the TODO's own text ("combine into a single enum") exactly.
2. The research confirms the blast radius is as small as it can possibly be: the *only* production code path (`pkg/flagutil/github.go`) never touches these fields, so it needs zero changes; the *only* other caller in the entire repo is a 2-call-site integration test.
3. `pkg/git/v2` is an internal package (`pkg/`, not a versioned/published module) with no external API stability contract to preserve.
4. A deprecation shim (Approach 2) would add real complexity to protect a compatibility need that, per the caller inventory, doesn't exist.

**Key Implementation Considerations**:
1. `SchemeType` as iota enum: `SchemeHTTPS` (zero value/default), `SchemeHTTP`, `SchemeSSH`.
2. Collapse the `if/else-if/else` at client_factory.go:314-329 into a `switch o.Scheme`.
3. Decide whether to keep a `WithScheme(SchemeType)` functional-option constructor or drop option-constructors entirely, since none are used in-repo today.
4. Add the missing `client_factory_test.go` covering all three scheme branches, including what happens with `CookieFilePath` set simultaneously (Gerrit priority).
5. Add the missing `http: true` case to `remote_test.go`.
6. Update `test/integration/test/moonraker_test.go`'s two call sites to `Scheme: git.SchemeHTTP` (or `git.WithScheme(git.SchemeHTTP)`).
7. Consider a `String()` method on `SchemeType` for debuggability.

**Testing Requirements**:
- Unit tests for `NewClientFactory` scheme selection (all three schemes, plus interaction with `CookieFilePath`)
- Unit test for `httpResolverFactory` `http:true` branch (closes existing gap)
- Update the 2 integration test call sites

**Migration/Rollout Strategy**: Atomic single PR — no phased rollout needed given the caller inventory above.

## Next Steps

(Action items will be added here)
