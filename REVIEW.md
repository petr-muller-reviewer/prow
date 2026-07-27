---
pr: 667
title: "Add helpers to reduce friction local development"
head_sha: 7301e1c860b921cddf787b92eeb5839b67bded4c
base: main
reviewed_at: 2026-04-03
verdict: APPROVE WITH SUGGESTIONS
---

# PR #667 — Add helpers to reduce friction in local development

Author: stmcginnis (Sean McGinnis) · +1149 / -13 · size/XXL · Multi-perspective maintainer review

## Recommendation: APPROVE WITH SUGGESTIONS

All three reviewers issued COMMENT verdicts — none requested changes. The PR
delivers meaningful DX improvements by building on existing integration test
infrastructure rather than creating a parallel system. Concerns raised are
real but non-blocking or addressable with minor fixes.

## Files changed

| File | What |
|---|---|
| `.gitignore` | Ignore `tilt-settings.yaml` and `tilt.d/*.tiltfile` |
| `Makefile` | New targets: `dev`, `dev-full`, `dev-tilt`, `dev-teardown` |
| `Tiltfile` | Tilt configuration for auto-rebuild/redeploy (Starlark) |
| `hack/dev-env.sh` | Main dev environment orchestrator |
| `hack/phony.sh` | Convenience wrapper for sending fake webhooks |
| `hack/tilt-apply-config.sh` | Applies Prow configmaps on config change |
| `hack/tilt-build.sh` | Builds a single Prow image for Tilt |
| `site/.../build-test-update.md` | Doc update: link to local dev + phony.sh |
| `site/.../getting-started-develop.md` | Doc update: link to local dev guide |
| `site/.../local-dev-tilt.md` | New doc: Tilt workflow guide |
| `site/.../local-dev.md` | New doc: local dev environment guide |
| `site/.../test/integration/_index.md` | Updated: adding-a-component instructions for core profile |
| `test/integration/lib.sh` | Core arrays + `$0`→`BASH_SOURCE` fix |
| `test/integration/setup-kind-cluster.sh` | Configurable host ports; containerd v2 registry config |
| `test/integration/setup-prow-components.sh` | `-profile=` flag; core/full dispatch |
| `tilt.d/.gitkeep` | Placeholder for personal Tiltfile overrides |

## Review checklist

- [ ] All findings reviewed and dispositioned
- [ ] Tested `make dev` on a clean checkout
- [ ] Tested `make dev` → `make dev-tilt` inner loop
- [ ] Tested `make dev-teardown`
- [ ] Verified `hack/phony.sh` sends a webhook successfully
- [ ] Verified `make integration` still passes (no regression from lib.sh / setup-kind-cluster.sh changes)
- [ ] Verified containerd registry approach works with CI kind version
- [ ] Docs render correctly in Hugo

## Converging concerns

Issues flagged independently by two or more reviewers. These carry the most weight.

### [Converging: 2+ reviewers] Phantom `-add=` flag reference in `lib.sh`
Reviewers: Code Quality, Maintainability

`test/integration/lib.sh:281` — Comment on `PROW_COMPONENTS_CORE` says
components "can be added individually with `hack/dev-env.sh -add=`", but
`dev-env.sh` has no `-add` flag.

Appears to be a leftover from an earlier design iteration. Remove the
reference or implement the flag.

### [Converging: 2+ reviewers] Triple source of truth for component definitions
Reviewers: Code Quality, Maintainability

Component definitions now live in three places that must be kept in sync:

- `test/integration/lib.sh` — `PROW_IMAGES`, `PROW_COMPONENTS`, `PROW_DEPLOYMENT_ORDER`
- `test/integration/lib.sh` — `PROW_COMPONENTS_CORE`, `PROW_DEPLOYMENT_ORDER_CORE`
- `Tiltfile` — `COMPONENT_DEFS`

The Starlark/bash split makes this unavoidable, but cross-reference comments
in each location would prevent silent drift. The `deck-tenanted` mismatch
(see below) is already an instance of this going wrong.

## Findings by severity

### [Bug] `deck-tenanted` inconsistency between Tiltfile and core profile
Reviewers: Code Quality, Maintainability

The Tiltfile unconditionally registers `deck-tenanted` as a workload
(`Tiltfile:218`) and applies `deck_tenant_deployment.yaml`. However,
`PROW_DEPLOYMENT_ORDER_CORE` in `test/integration/lib.sh` does *not* include
`deck_tenant_deployment.yaml` or `WAIT_deck-tenanted`.

**Impact**: `make dev` + `tilt up` will have Tilt trying to manage a workload
that doesn't exist in the cluster, causing errors or an unexpected deployment.

**Fix**: either add `deck_tenant_deployment.yaml` + its wait to
`PROW_DEPLOYMENT_ORDER_CORE`, or gate the Tiltfile's deck-tenanted
registration behind the `extra_components` mechanism.

### [Bug] Tiltfile `prow_component` doc/code mismatch
Reviewers: Code Quality, Maintainability

`site/content/en/docs/local-dev-tilt.md:135-140` shows this example for
personal Tiltfile overrides:

```
# tilt.d/narrow-hook-deps.tiltfile
prow_component('hook', ['cmd/hook/', 'pkg/github/', 'pkg/plugins/'])
```

But the actual function at `Tiltfile:184` only accepts a single `name`
argument:

```
def prow_component(name):
    ...
```

Anyone following the docs gets a runtime error from Tilt. Either add an
optional `override_src_dirs` parameter, or change the example to modify
`COMPONENT_DEFS` directly.

### [Suggestion] `hack/phony.sh` — silent failure and hardcoded context
Reviewers: Code Quality

**Silent HMAC failure** — `hack/phony.sh:24`: If `kubectl` fails (cluster not
running, secret missing), `base64 -d` silently produces empty output. The
script proceeds with an empty HMAC, resulting in a confusing "unauthorized"
error instead of a clear message.

```
HMAC=$(kubectl --context=kind-kind-prow-integration \
    get secret hmac-token -o jsonpath='{.data.hmac}' | base64 -d) || {
  echo >&2 "Failed to read HMAC token. Is the dev cluster running? (make dev)"
  exit 1
}
```

**Hardcoded context** — Uses `kind-kind-prow-integration` literally instead
of sourcing `lib.sh` and using `_KIND_CONTEXT` / `do_kubectl`, unlike the
other new scripts. Would break if cluster name ever changes.

### [Suggestion] Shared cluster name between dev and integration environments
Reviewers: Deployment Risk, Maintainability

Dev and integration environments share the kind cluster name
`kind-prow-integration`. Running `make dev` creates the cluster with ports
8080/8443; a subsequent `make integration` sees the cluster as running,
skips creation, then tests fail reaching services on port 80.

The docs warn about this, but the failure mode is not graceful — no error
message, just confusing test failures. Consider:

- Using a separate cluster name for dev, or
- Making `setup-kind-cluster.sh` port-aware when reusing an existing cluster

### [Suggestion] `PROW_DEPLOYMENT_ORDER_CORE` duplication in `lib.sh`
Reviewers: Maintainability

`PROW_DEPLOYMENT_ORDER_CORE` (~80 lines) is a near-complete copy of the first
~75% of `PROW_DEPLOYMENT_ORDER`. Changes to infrastructure entries (CRDs,
secrets, ingress, fakes) or YAML file renames require updating both arrays in
lockstep.

Not blocking for this PR, but a follow-up could tag entries and derive the
core list programmatically, or compose the deployment order from reusable
blocks (infrastructure, core-fakes, core-services, extra-services). At
minimum, add cross-reference comments in both arrays.

### [Suggestion] Double argument parsing loop in `setup-prow-components.sh`
Reviewers: Maintainability

`test/integration/setup-prow-components.sh` — `-profile=` is parsed in a
first pass over all arguments, then all arguments are parsed again in a
second pass. Functionally correct, but unusual — a future contributor might
add new flags to only one loop. A comment explaining the two-pass rationale
would help.

### [Deploy] Containerd registry config change affects `make integration`
Reviewers: Deployment Risk

`test/integration/setup-kind-cluster.sh` replaces the deprecated inline
`registry.mirrors` containerd config with `config_path =
"/etc/containerd/certs.d"` + `hosts.toml` files written via `docker exec`.

This is the officially recommended approach and required for containerd
v2.2+, but it applies to **all** `setup-kind-cluster.sh` invocations,
including `make integration`.

- Verify CI kind image ships containerd ≥ 1.5
- Developers with pre-existing kind clusters must `teardown.sh -kind-cluster`
  and recreate — the new `hosts.toml` files are ignored without
  `config_path` in containerd config. This is not documented.

### [Nit] `trap` quoting in `tilt-build.sh`
Reviewers: Code Quality

`hack/tilt-build.sh:52`:

```
trap "rm -f ${tmpfile}" EXIT
```

Expands `${tmpfile}` at definition time. Matches the existing pattern in
`setup-prow-components.sh` (with `# shellcheck disable=SC2064`), but the
disable comment is missing here. Add it to signal intent.

## Positive observations

### [Good] Highlights (all three reviewers agree)

- **Reuses integration test infra** — builds on existing
  `test/integration/` scripts rather than duplicating cluster setup logic.
  Fixes to integration tests automatically benefit the dev environment.
- **`BASH_SOURCE` fix** in `lib.sh` — correct and necessary since the file is
  `source`d from scripts in different directories. Produces identical
  results for all existing callers.
- **Safe defaults** — `_PROFILE="full"` preserves existing `make integration`
  behavior. New flags default to pre-existing values (80, 443).
- **Unprivileged ports** — defaulting to 8080/8443 avoids the root
  requirement on Linux.
- **Extensibility** — `tilt-settings.yaml` overrides and `tilt.d/` personal
  Tiltfiles are well-designed escape hatches.
- **`allow_k8s_contexts`** in the Tiltfile prevents accidental deployment to
  a production cluster.
- **`print_connection_info`** — excellent UX; gives developers
  copy-pasteable commands for every common task.
- **No security concerns** — no hardcoded credentials; HMAC tokens read from
  the cluster at runtime; local registry is HTTP-only on localhost.
- **Comprehensive docs** — both `local-dev.md` and `local-dev-tilt.md` are
  thorough and well-organized.
- **Forward-compatible containerd config** — adopts the kind-recommended
  `hosts.toml` approach required for containerd v2.2+.

## PR comment draft

> This is a solid PR that delivers real developer experience value by
> building on existing infrastructure rather than creating a parallel
> system. Approving with suggestions. The main things I'd like to see
> addressed (non-blocking, can be follow-up): fix the incorrect
> `prow_component` multi-arg example in the docs, remove the stale `-add=`
> flag reference in `lib.sh`, and consider gating the `deck-tenanted`
> workload in the Tiltfile on the active profile to avoid confusing Tilt
> errors in core-only setups. The containerd registry config change is the
> right approach but please verify CI kind image compatibility before
> merging.

---

Multi-perspective maintainer review generated 2026-04-03 · Reviewers: Code
Quality, Maintainability, Deployment Risk + Advisor synthesis · PR:
kubernetes-sigs/prow#667
