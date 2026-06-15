---
pr: 573
title: "Adding option to enable Back End HTTPS for Prow Ingress"
head_sha: 814fb94b8d3379b2bef43f54564b0ecd3d97a32b
base: main
reviewed_at: "2026-06-15T16:20:00Z"
verdict: REQUEST_CHANGES
refresh_log:
  - previous_sha: 53a81071b1f5a432c33362a42aa7bc9837f7ed14
    new_sha: 814fb94b8d3379b2bef43f54564b0ecd3d97a32b
    summary: "PR rebased onto newer main (force-push); no changes to PR-specific code. One author comment explaining motivation (Fidelity Investments corporate policy requires backend HTTPS on ingresses)."
---

# PR #573 — Adding option to enable Back End HTTPS for Prow Ingress

**Author:** NiJuFirenzia · **+479 / -13** · Addresses [#328](https://github.com/kubernetes-sigs/prow/issues/328)
**Files:** `cmd/deck/`, `cmd/hook/`, `pkg/flagutil/ssl.go`

## Recommendation: REQUEST_CHANGES

All three independent reviewers converge on the same core issues: the `--client-cert-file` flag is semantically misleading (it's a CA cert, not a client cert for mTLS), and the `AppendCertsFromPEM` return value is silently ignored, producing cryptic failures in production. While deployment risk is low (purely opt-in), this PR introduces TLS infrastructure that operators will rely on for security-critical paths — naming precision and fail-fast behavior are essential. The required changes are straightforward and should not need significant rework.

## Reviewer Perspectives

| Perspective | Verdict | Detail |
|---|---|---|
| Code Quality | REQUEST_CHANGES | Unchecked `AppendCertsFromPEM` return, misleading flag naming, incorrect error messages |
| Maintainability | REQUEST_CHANGES | MEDIUM burden — naming confusion will cause ongoing support costs; TLS client logic trapped in cmd/deck |
| Deployment Risk | LOW RISK | Purely opt-in via new flags. Zero behavioral change for existing deployments. |

## Converging Concerns

Issues flagged independently by 2+ reviewers carry the highest confidence:

- `--client-cert-file` is misnamed — it's a CA cert, not a client cert [CQ, MT, DR]
- `AppendCertsFromPEM` return value silently ignored — cryptic runtime failures [CQ, MT]
- Validation over-couples server TLS and hook CA cert; error message is incorrect [CQ, MT, DR]
- No `os.Stat` file existence check in `SSLOptions.Validate` [CQ, MT]
- `sslEnabledSchema` should be `sslEnabledScheme` [CQ, MT]

## Activity Since Previous Review

Since previous review (2026-06-01):

- **Force-push (rebase)**: PR rebased onto newer main (`53a81071b` → `814fb94b8`). The PR-specific code in `cmd/deck/`, `cmd/hook/`, and `pkg/flagutil/ssl.go` is unchanged — same 2 commits (`2ad6fa101` initial implementation, `814fb94b8` removing `--enable-ssl` boolean).
- **Author comment (2026-06-12)**: @NiJuFirenzia explained the motivation — Fidelity Investments has a corporate policy requiring backend HTTPS on all ingresses. Without this feature, the ingress cannot connect to deck/hook pods over HTTP. Author agreed to make changes and add documentation.
- No new reviews or inline comments since the review.
- All prior findings remain unaddressed in code.

## Prior Review Context

Reviews from **@ivankatliarchuk** (changes requested) and **@petr-muller** (commented). Key points already raised:

- `--client-cert-file` naming confusion (CA cert vs. client cert / mTLS)
- Dead `cert` field on `helpAgent`
- Coupling between `--server-cert-file` and `--client-cert-file`
- Renamed `SSLEnablementOptions` → `SSLOptions`, removed `--enable-ssl` boolean (done in latest push)
- `sslEnabledSchema` → should be `Scheme`
- Missing file existence checks in `Validate`

## Review Checklist

### Required Changes

- [ ] **blocker** Rename `--client-cert-file` to `--hook-ca-cert-file` [CQ, MT, DR]
- [ ] **blocker** Check `AppendCertsFromPEM` return value; fail if no certs parsed [CQ, MT]
- [ ] **blocker** Decouple server TLS from hook CA cert validation; fix error message [CQ, MT, DR]
- [ ] **major** Add `os.Stat` file existence checks in `SSLOptions.Validate` [CQ, MT]
- [ ] **major** Fix `sslEnabledSchema` → `sslEnabledScheme` [CQ, MT]

### Additional Issues

- [ ] **major** Dead `cert` field on `helpAgent` — set but never read
- [ ] **major** Test uses `"dummy cert content"` (not valid PEM) — fragile once return check is added [CQ]
- [ ] **minor** Rename `isHttpsPath` → `isHTTPSPath` (Go initialism convention) [CQ, MT]
- [ ] **minor** Use `&http.Client{}` instead of `http.DefaultClient`
- [ ] **minor** Direct unit tests for `isHttpsPath` (edge cases)
- [ ] **nit** Comment style: `//Set` → `// Set`
- [ ] **nit** Import grouping in `pluginhelp_test.go`
- [ ] **nit** Test naming convention (underscored PascalCase vs. repo style)
- [ ] **nit** Unrelated blank line removal in `cmd/hook/main.go`
- [ ] **nit** `Status_Code_Error` test sets both resp and err (unusual mock state)

## Detailed Findings

### `--client-cert-file` is misleading — it's a CA cert, not a client cert (blocker) [CQ, MT, DR]

`cmd/deck/pluginhelp.go:55-65`

The cert loaded from `--client-cert-file` is used as a `RootCAs` entry to verify hook's server certificate. This is one-way TLS (deck trusts hook), **not** mutual TLS (deck authenticating itself to hook).

```go
caCertPool := x509.NewCertPool()
caCertPool.AppendCertsFromPEM(caCert)
client = &http.Client{
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{
            RootCAs: caCertPool,  // this is a CA trust pool
        },
    },
}
```

An operator reading `--client-cert-file` would expect mTLS. Rename to `--hook-ca-cert-file` or `--hook-tls-ca-file`.

### `AppendCertsFromPEM` return value silently ignored (blocker) [CQ, MT]

`cmd/deck/pluginhelp.go:61`

`x509.CertPool.AppendCertsFromPEM()` returns a `bool` indicating whether any certificates were parsed. If the file content is not valid PEM (corrupted, wrong format), the cert pool is silently empty and all TLS connections to hook fail with a cryptic handshake error.

```go
if !caCertPool.AppendCertsFromPEM(caCert) {
    return nil, fmt.Errorf("failed to parse any certificates from %s", cert)
}
```

The test at `pluginhelp_test.go:89` using `"dummy cert content"` will also need to use valid PEM once this check is added.

### Validation over-couples server TLS and hook CA cert (blocker) [CQ, MT, DR]

`cmd/deck/main.go:177-179`

```go
if o.ssl.CertFile != "" && o.clientCertFile == "" {
    return errors.New("flag --server-cert-file and --server-key-file was set to true but required flag --client-cert-file was not set")
}
```

**Two problems:**

1. **Logically wrong**: Deck serving HTTPS to ingress is independent of whether deck calls hook over HTTPS. An operator may want TLS on deck without configuring the hook CA cert (hook behind a separate proxy, or `--hook-url` not set). The check on lines 167-174 (triggered when `--hook-url` is HTTPS) already handles the real requirement.

2. **Error message is incorrect**: Says "was set to true" but these are string flags, not booleans. Should be "were set" and drop "to true". (Moot if the check is removed.)

### No file existence check in `SSLOptions.Validate` (major) [CQ, MT]

`pkg/flagutil/ssl.go:39-48`

Other validators in this codebase (e.g. `kubernetes_cluster_clients.go`) check file existence at startup via `os.Stat`. Without it, the operator gets a cryptic TLS failure from `ListenAndServeTLS` instead of a clear message.

```go
if _, err := os.Stat(o.CertFile); err != nil {
    return fmt.Errorf("--server-cert-file: %w", err)
}
if _, err := os.Stat(o.KeyFile); err != nil {
    return fmt.Errorf("--server-key-file: %w", err)
}
```

### Dead `cert` field on `helpAgent` (major)

`cmd/deck/pluginhelp.go:40`

The `cert` field is assigned in `newHelpAgent` but never read afterward — the TLS client is fully constructed during initialization. Remove it to avoid confusion about whether cert rotation is supported (it isn't).

### Go naming: `isHttpsPath` and `sslEnabledSchema` (minor) [CQ, MT]

`cmd/deck/pluginhelp.go:36, 105`

Go convention for initialisms is all-caps: `isHTTPSPath`. The constant `sslEnabledSchema` should be `sslEnabledScheme` — a URL has a *scheme*, not a *schema*.

### `http.DefaultClient` shared reference (minor)

`cmd/deck/pluginhelp.go:69`

Assigning `http.DefaultClient` directly means any future modification to the agent's client (timeouts, middleware) would mutate the global default. Use `&http.Client{}` instead.

### Missing direct tests for `isHttpsPath` (minor)

`cmd/deck/pluginhelp.go:105-111`

The function is tested indirectly. A direct table-driven test would cover edge cases: empty string, uppercase `HTTPS`, non-HTTP schemes (`ftp://`), bare paths without scheme.

### Comment style: missing space after `//` (nit)

`cmd/deck/pluginhelp.go:49`

```
//Set custom http client if ssl is enabled
```

Should be `// Set custom http client if ssl is enabled` per Go style.

### Import grouping in `pluginhelp_test.go` (nit)

`cmd/deck/pluginhelp_test.go:24-31`

`sigs.k8s.io/prow/pkg/pluginhelp` is mixed in with stdlib imports. Convention is stdlib, then a blank line, then external imports.

### Test naming convention (nit)

`cmd/deck/pluginhelp_test.go`

Test case names use `PascalCase_With_Underscores` (e.g. `Invalid_Url_Returns_Error`). Existing tests in this repo use lowercase with spaces. Consider following the existing convention for consistency.

### Unrelated formatting change in `cmd/hook/main.go` (nit)

`cmd/hook/main.go:97`

A blank line between the `AddFlags` loop and the `webhookSecretFile` registration was removed. This is unrelated to the feature and adds noise to the diff.

### `Status_Code_Error` test has unusual mock state (nit)

`cmd/deck/pluginhelp_test.go:171-173`

The test sets both `mockResp` (with status 500) and `mockErr`. In real HTTP, when `RoundTrip` returns an error the response is typically nil. This creates an impossible state.

## Deployment Notes

- When enabling TLS on deck or hook, operators **must** update readiness/liveness probe configurations to use HTTPS simultaneously.
- If `--hook-url` uses an HTTPS scheme, the (renamed) `--hook-ca-cert-file` flag becomes required for deck to communicate with hook.
- No action required for operators who do not set any of the new flags — behavior is identical to the current release.
- `newHelpAgent` failure is fatal — a bad cert file prevents deck from starting entirely, not just degrading the plugin-help endpoint.

## What Looks Good

- `SSLOptions` flagutil is a clean, reusable `OptionGroup` abstraction following existing patterns (`BugzillaOptions`, `GitHubOptions`)
- Conditional `ListenAndServeTLS` / `ListenAndServe` branching follows existing patterns (`fakegitserver`, `webhook-server`)
- Good test coverage for the new flagutil, `newHelpAgent`, and `getHelp` including cache behavior
- The removed `--enable-ssl` boolean (latest commit) simplifies the interface — presence of cert/key files implies intent
- `newHelpAgent` returning an error instead of hiding failure improves debuggability
- Caching logic with proper mutex locking was correctly preserved when adding the custom HTTP client

## PR Comment Draft

Thanks for adding TLS support to deck and hook — the overall approach using `SSLOptions` as a reusable flagutil is clean, and the opt-in design keeps deployment risk low. However, I'm requesting changes before merge. The `--client-cert-file` flag is misleadingly named — it's actually a CA certificate for verifying hook's server cert, not a client cert for mTLS. This needs to be renamed (e.g., `--hook-ca-cert-file`). Additionally, the `AppendCertsFromPEM` return value must be checked to avoid cryptic runtime failures, the validation logic in `main.go` has an incorrect error message and unnecessarily couples server TLS with hook CA config, and `SSLOptions.Validate` should verify file existence at startup. These are all targeted fixes that shouldn't require major rework. Happy to re-review once addressed.
