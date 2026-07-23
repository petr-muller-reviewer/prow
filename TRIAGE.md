---
issue: kubernetes-sigs/prow#589
title: "TODO (listx): Combine HTTPS, HTTP, and SSH schemes into a single enum."
state: open
labels: lifecycle/stale
main_sha: 0633879af8026d056e1a5dbe1e29f5a98f6acec3
triaged_at: 2026-07-23T10:26:35Z
verdict: accepted
---

## Findings

### [cause] Tri-state boolean anti-pattern with unenforced precedence
- detail: `ClientFactoryOpts` uses two independent `*bool` fields to represent three mutually exclusive schemes. The doc comment claims `UseInsecureHTTP` is "overridden" by SSH, but this is never validated — if both are true, SSH silently wins and `UseInsecureHTTP` is never read at all.
- evidence: `pkg/git/v2/client_factory.go:105-110` (fields + comment), `pkg/git/v2/client_factory.go:314-329` (branch where `UseSSH` check at line 315 makes the `else` branch containing `UseInsecureHTTP` at line 325 unreachable when both are true).

### [related-code] TODO comment
- where: `pkg/git/v2/client_factory.go:107`
- excerpt: |
    // TODO (listx): Combine HTTPS, HTTP, and SSH schemes into a single enum.
    UseInsecureHTTP *bool
    // UseSSH, defaults to false
    UseSSH *bool

### [related-code] Scheme selection logic in NewClientFactory
- where: `pkg/git/v2/client_factory.go:314-329`
- excerpt: |
    if o.UseSSH != nil && *o.UseSSH {
        remote = &sshRemoteResolverFactory{host: o.Host, username: o.Username}
    } else if o.CookieFilePath != "" {
        remote = &gerritResolverFactory{}
    } else {
        remote = &httpResolverFactory{host: o.Host, http: o.UseInsecureHTTP != nil && *o.UseInsecureHTTP, username: o.Username, token: o.Token}
    }

### [related-code] Scheme resolution in httpResolverFactory
- where: `pkg/git/v2/remote.go:132-137`
- excerpt: |
    func (f *httpResolverFactory) resolve(org, repo string) (string, error) {
        scheme := "https"
        if f.http {
            scheme = "http"
        }

### [related-code] Caller inventory (blast radius of a breaking change)
- where: `pkg/flagutil/github.go:339-360`
- excerpt: |
    gitv2.ClientFactoryOpts{Censor: ..., CookieFilePath: ..., Host: ..., Persist: ...}
    // never sets UseSSH or UseInsecureHTTP
    gitClientFactory, err := gitv2.NewClientFactory(opts.Apply)
- relevance: the sole production caller of `NewClientFactory`/`Apply` never sets either boolean field, so removing them is a no-op change for production code.

### [related-code] Only other caller in the repo
- where: `test/integration/test/moonraker_test.go:148,382`
- excerpt: |
    o.UseInsecureHTTP = &trueVal
- relevance: the only other place in the entire repository that sets `UseInsecureHTTP`/`UseSSH` (grep-confirmed repo-wide); an enum refactor needs to update exactly these two lines. No other in-repo caller exists.

### [related-code] Test coverage gaps
- where: `pkg/git/v2/remote_test.go` (TestHTTPResolverFactory, TestHTTPResolverFactory_NoAuth)
- detail: every existing test leaves `httpResolverFactory.http` at its zero value (`false`), so the `"http"` scheme branch (`remote.go:135`) is never exercised. `pkg/git/v2/client_factory_test.go` does not exist at all, so `NewClientFactory`'s scheme-selection branch (SSH vs Gerrit vs HTTP(S) priority) has zero direct test coverage.

### [related-issue] Maintainer sign-off already given
- ref: kubernetes-sigs/prow#589 (comments)
- relevance: maintainers `matthyx` (2026-01-12) and `petr-muller` (2026-01-16) both told the author to proceed with an implementation; no objection or alternative direction was raised.

## Checked
- Verified the TODO comment and `ClientFactoryOpts` struct are unchanged in current upstream `main` (`0633879af`) versus when the issue was filed.
- Repo-wide grep for `UseInsecureHTTP`, `UseSSH`, `WithInsecureHTTP`, `WithSSH`, and `.Apply(` on `ClientFactoryOpts` — confirmed only 3 files reference these at all.
- Read `NewClientFactory`, `httpResolverFactory`, `sshRemoteResolverFactory`, `gerritResolverFactory` in full (`pkg/git/v2/client_factory.go`, `pkg/git/v2/remote.go`).
- Read `pkg/git/v2/remote_test.go` in full to assess existing coverage.
- Searched `pkg/git/v2`, `pkg/git` (v1), `pkg/flagutil`, `pkg/gerrit`, and repo-wide for an existing iota/enum pattern to mirror — none found; this would be a novel (but idiomatic) addition.
- Reviewed all issue comments and timeline events: `/assign` by author (2026-01-12), maintainer replies (2026-01-12, 2026-01-16), two `lifecycle/stale` applications by `k8s-triage-robot` (2026-04-16, 2026-07-16) with one `/remove-lifecycle stale` by the author in between (2026-04-17).

## Next steps
- Post an augmentation comment adding root-cause framing, the caller-inventory scope check, and an implementation sketch (drafted, not yet posted per maintainer decision).
- Apply labels: `/area git`, `/kind cleanup`, `/help-wanted`.
- Separately decide whether to check in with assignee `tsj-30` — self-assigned 2026-01-12, no PR six-plus months later, issue currently `lifecycle/stale` (reapplied 2026-07-16).

## Open questions
- @tsj-30 are you still planning to work on this, or should the assignment be opened back up?
- Given only one in-repo caller sets these fields today, is a straight breaking change (no deprecation shim) acceptable, or would maintainers prefer a transition period regardless of the small blast radius?
