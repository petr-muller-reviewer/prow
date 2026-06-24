---
issue: 51
title: "Restrict Prow for Users running in Environments with OPA constraints"
url: https://github.com/kubernetes-sigs/prow/issues/51
author: NiJuFirenzia
created_at: 2023-04-06
triaged_at: 2026-06-24T00:04:02Z
state: closed
closed_at: 2026-06-22T20:20:56Z
verdict: LEGITIMATE
effort: 2
labels_current:
  - lifecycle/rotten
labels_proposed:
  - area/pod-utilities
  - kind/feature
  - help-wanted
  - remove-lifecycle stale
retitle: "Support Pod Security Standards (restricted profile) for ProwJob pods and control plane"
refresh_log:
  - triaged_at: 2026-05-02
    summary: Initial triage
  - triaged_at: 2026-06-01T00:00:38Z
    summary: "Label escalated lifecycle/stale → lifecycle/rotten by k8s-triage-robot on 2026-05-23; no human activity, no linked PRs"
  - triaged_at: 2026-06-24T00:04:02Z
    summary: "Auto-closed by k8s-triage-robot on 2026-06-22 (lifecycle/rotten 30d expiry). No human activity. /reopen to restore."
---

# Triage: Issue #51

**Issue**: [#51 — Restrict Prow for Users running in Environments with OPA constraints](https://github.com/kubernetes-sigs/prow/issues/51)
**Verdict**: LEGITIMATE · **Kind**: Feature · **Effort**: Level 2 (Moderate) · **Label**: help-wanted

**Since previous triage (2026-05-02)**:
- 2026-05-23: `k8s-triage-robot` escalated `lifecycle/stale` → `lifecycle/rotten`.
- 2026-06-22: `k8s-triage-robot` auto-closed the issue as "Not Planned" (lifecycle/rotten 30d expiry). This is the third bot closure; the author reopened twice before (2025-01-02, 2026-01-09).
- No human comments, no linked PRs. Analysis unchanged. Issue can be reopened with `/reopen` and the augmentation comment posted simultaneously.

---

## Overview

The issue requests OPA/Gatekeeper compatibility for Prow. The core finding: **Prow doesn't actually need privileges**. No hardcoded privileged mode, no docker.sock mounts, no root requirements in Go code. The gap is missing SecurityContext boilerplate — Prow predates Pod Security Standards and never added it.

**Proposed retitle**: "Support Pod Security Standards (restricted profile) for ProwJob pods and control plane"

---

## The Two Gaps

### Gap 1: ProwJob Pods — No Container-Level SecurityContext

Utility containers (clonerefs, initupload, entrypoint, sidecar) are injected by `pkg/pod-utils/decorate/podspec.go` without container-level SecurityContext. PSS "restricted" requires each container to set:

- `allowPrivilegeEscalation: false`
- `capabilities.drop: [ALL]`
- `seccompProfile.type: RuntimeDefault`

`DecorationConfig` (`pkg/apis/prowjobs/v1/types.go:564-576`) only exposes `RunAsUser`, `RunAsGroup`, `FsGroup` — none of the container-level fields PSS needs.

### Gap 2: Control Plane Pods — Bare Starter Manifests

All starter and integration deployment manifests ship with zero SecurityContext:

- `config/prow/cluster/starter/starter-gcs.yaml`
- `test/integration/config/prow/cluster/{deck,hook,crier,horologium,tide}_deployment.yaml`

Missing from all: `runAsNonRoot`, `allowPrivilegeEscalation`, `capabilities.drop`, `seccompProfile`.

---

## Key Findings

- **Core code is clean**: No `privileged: true` in Go code, no docker.sock, no hardcoded root. `decorateSpec()` is non-overriding.
- **Utility containers are filesystem-safe**: All writes go to volume mounts (`/logs`, `/tools`, `/home/prow/go`), so `readOnlyRootFilesystem: true` should work — needs a verification audit.
- **Privileged only in tests**: `privileged: true` appears only in Maistra test configs; `allowPrivilegeEscalation` only in nginx test. Not in Prow's own deployments.
- **No policy integration**: Admission webhook validates spec immutability only. No OPA/Gatekeeper wiring, no PSS enforcement, no e2e tests in restricted namespaces.

---

## Recommended Solution

**Opt-in SecurityContext extension + hardened starter manifests**

Extend DecorationConfig with the missing SecurityContext fields (following the existing RunAsUser/RunAsGroup/FsGroup pattern), plumb them through `decorateSpec()` to container-level contexts on all four utility containers, and update starter manifests with PSS "restricted" defaults. All new fields are optional pointers — nil means "don't set", so existing configs are unaffected.

**Pros**: Zero breaking changes · follows established pattern · starter manifests give new deployments secure defaults  
**Cons**: Not secure by default for ProwJob pods (operators must opt in)

### PR Decomposition

1. **(easiest, pure YAML)** Harden starter manifests with PSS "restricted" SecurityContext
2. **(API)** Extend DecorationConfig: `RunAsNonRoot`, `AllowPrivilegeEscalation`, `ReadOnlyRootFilesystem`, `SeccompProfile`, `Capabilities`
3. **(plumbing)** Apply container-level SecurityContext to utility containers in `decorateSpec()`
4. **(docs)** "Running Prow in PSS-restricted environments" guide

---

## Effort Assessment: Level 2 (Moderate)

| Factor | Assessment | Level |
|--------|-----------|-------|
| Scope | ~8-12 files, ~200-400 LOC | 2-3 |
| Complexity | Mechanical field plumbing, one FS audit | 1-2 |
| Expertise | K8s SecurityContext + Prow decoration flow | 2-3 |
| Clarity | Standardized API, no design ambiguity | 1-2 |
| Testing | Unit tests follow existing patterns | 2-3 |
| Backwards Compat | Fully compatible (optional pointer fields) | 1-2 |
| Architecture | Perfect fit — extends existing pattern | 1-2 |
| External Deps | None | 1-3 |

---

## Key Code References

| File | What |
|------|------|
| `pkg/pod-utils/decorate/podspec.go:831-844` | SecurityContext application in `decorateSpec()` |
| `pkg/apis/prowjobs/v1/types.go:564-576` | DecorationConfig fields (RunAsUser, RunAsGroup, FsGroup) |
| `pkg/config/config.go:676-707` | DefaultDecorationConfigs (global defaults, org/repo filters) |
| `podspec.go:494-505, 569-579, 649-661, 937-950` | Utility container creation (no SecurityContext today) |
| `pkg/clonerefs/run.go:169,220` | clonerefs file writes (volume mounts) |
| `cmd/entrypoint/main.go:48-51` | entrypoint dir creation (volume mounts) |
| `config/prow/cluster/starter/starter-gcs.yaml` | Starter manifests (no SecurityContext) |
| `cmd/admission/admission.go` | Admission webhook (no security validation) |

---

## Proposed GitHub Comment

```
/retitle Support Pod Security Standards (restricted profile) for ProwJob pods and control plane

Prow's core code does not actually require elevated privileges — it doesn't hardcode privileged mode, doesn't mount `docker.sock`, and doesn't run as root. The DecorationConfig already supports configurable `RunAsUser`, `RunAsGroup`, and `FsGroup` via `pkg/apis/prowjobs/v1/types.go`. However, Prow currently cannot pass Pod Security Standards (PSS) "restricted" profile admission because of two gaps:

1. **ProwJob pods**: The utility containers injected by decoration (clonerefs, initupload, entrypoint, sidecar) in `pkg/pod-utils/decorate/podspec.go` are created without container-level SecurityContext. PSS "restricted" requires each container to explicitly set `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, and `seccompProfile.type: RuntimeDefault`. The DecorationConfig has no fields for these container-level settings.

2. **Control plane pods**: The starter deployment manifests (`config/prow/cluster/starter/`) ship without any SecurityContext — no `runAsNonRoot`, no capability drops, no seccomp profile. In a PSS-restricted namespace, these deployments would be rejected by admission.

The fix would extend DecorationConfig with additional SecurityContext fields (following the existing pattern for `RunAsUser`/`RunAsGroup`/`FsGroup`) and apply them to both pod-level and container-level contexts during decoration. Starter manifests should also be updated with PSS "restricted"-compatible defaults. This can be done in a backwards-compatible way since all new fields would be optional — nil means "don't set". The work splits naturally into independent PRs: manifest hardening, DecorationConfig extension, container-level SecurityContext plumbing, and documentation. If you have specific OPA constraints you're hitting beyond what PSS covers, sharing those details would help scope this further.

/area pod-utilities
/kind feature
/help-wanted
/remove-lifecycle stale

<details>
<summary>Triage information</summary>

This comment was made by experimental [Claude triage helper](https://github.com/petr-muller/prow/blob/claude-maintenance-helpers/.claude/skills/maintenance-issue-triage/SKILL.md). I reviewed the content and I hope it is useful and not AI slop. If you have feedback please reach out to me.

Full triage: https://github.com/petr-muller/prow/blob/issue-triage-51/ISSUE-TRIAGE.md
</details>
```
