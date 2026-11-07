---
pr: kubernetes-sigs/prow#1064
title: "chore(deps): bump go.yaml.in/yaml/v3 from 3.0.5 to 3.0.6"
head_sha: 93d6d95895d8e451cbb2e60e960d737d89d99ddf
base: main
reviewed_at: 2026-11-07T21:10:33Z
verdict: approve
---

# Review

## Verdict

Approve. A single direct Go module moves 3.0.5 → 3.0.6, the release contains one upstream fix that lands exactly on a path we exercise (`pkg/genyaml` encodes YAML), it is 25 days old with no advisories on either side, and everything we build and test still passes.

## What this PR does

- Bumps the direct dependency `go.yaml.in/yaml/v3` from v3.0.5 to v3.0.6 in `go.mod:61`; `go.sum` changes only the module hash (the `/go.mod` hash is identical, consistent with an unchanged `go.mod` inside the release).
- Nothing else changes: `git diff --stat` is `go.mod | 2 +-`, `go.sum | 4 ++--` — no vendor tree, no project source, no docs. Dep-only.
- v3.0.6 contains exactly one upstream commit, `5a62aba6` "resolve: hint-class literal merge-key string `<<` so it's quoted on encode": one line in `resolve.go` (`t[int('<')] = 'M'`) plus a 37-line regression test.
- That fixes a round-trip bug where `Marshal` of an ordinary Go map with the literal key `<<` emitted it unquoted, and `Unmarshal` then read it as a YAML merge directive — splicing the value's keys into the parent map and dropping the `<<` entry.

## Findings

None.

## Checked

- Freshness: v3.0.6 published 2026-10-13T18:50:51Z (25 days before review); tagged release (proxy `Origin.Ref` = `refs/tags/v3.0.6`), not a pseudo-version.
- OSV: no advisories for `go.yaml.in/yaml/v3` at v3.0.5 or v3.0.6; also none under the legacy `gopkg.in/yaml.v3` path for either version.
- `govulncheck -scan module`: base (`9f4d1868e`) and head both exit 0 with zero findings, so no symbol-level scan was warranted. (`govulncheck` v1.9.0 rejects module scanning at the repo root, which has no `.go` files; it was run from `pkg/genyaml` in the same module.)
- Usage (direct, 4 files): `pkg/genyaml/genyaml.go:188,194,483,508` (Unmarshal/Marshal/Encoder, the encode path the fix touches), `pkg/genyaml/genyaml_test.go`, `pkg/plugins/testfreeze/checker/checker.go:340` (Unmarshal of prow config), `hack/ts-rollup/main.go:84` (tooling Unmarshal).
- Blast radius of the behavior change: `resolveMap` has no other `<`-prefixed entry besides `<<`, and nothing in our tree (Go or YAML config) uses a literal `<<` key, so the only effect is that such a key would now be quoted correctly.
- Decode-only users (`testfreeze/checker`, `hack/ts-rollup`) are unaffected — the change is in the encoder hint table.
- `go build ./pkg/genyaml/... ./pkg/plugins/testfreeze/... ./hack/ts-rollup/...` succeeds; `go test ./pkg/genyaml/... ./pkg/plugins/testfreeze/...` passes.

## Open questions

None.

## Dependency followups

No improvement opportunities identified.

- Examined: `go.yaml.in/yaml/v3` v3.0.5 → v3.0.6 (direct; the only module that changes — `go.sum` moves no other entry, so there is no transitive churn). Upstream range v3.0.5...v3.0.6 is a single commit, `5a62aba6` (merge-key `<<` quoting fix) plus its regression test.
- No deprecations in the range (no `Deprecated:` markers anywhere in v3.0.6), no new or superseding APIs, no new features, no behavior/default changes beyond the fixed quoting. Our call sites (`pkg/genyaml`, `pkg/plugins/testfreeze/checker`, `hack/ts-rollup`) need no adaptation.
- Noted but deliberately not recorded as a followup: upstream keeps the v3 line frozen for security fixes only, and `go.yaml.in/yaml/v4` reached v4.0.0 on 2026-11-04 (3 days old, unsoaked). A v3 → v4 migration is not unlocked by this bump and should at minimum wait for v4 to soak; revisit separately if it is wanted.
