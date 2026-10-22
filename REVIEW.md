---
pr: kubernetes-sigs/prow#902
title: "chore(deps): bump github.com/google/ko from 0.18.1 to 0.19.1 in /hack/tools"
head_sha: 00ca0969b4245e14b873439e4c8cb4513542b8cb
base: main
reviewed_at: 2026-09-01T00:01:19Z
verdict: approve
---

## What this PR does

- Dependabot bump of `github.com/google/ko` `v0.18.1 → v0.19.1` in the separate `hack/tools` go module (build tooling, not the shipped Prow binaries).
- Only `hack/tools/go.mod` and `hack/tools/go.sum` change; no project source touched. Dep-only PR.
- Drags a large sweep of transitive/indirect version bumps in `ko`'s own dependency graph (cosign v2→v3, docker/moby, sigstore family, go-openapi, otel, etc.) — normal `go mod tidy` churn from the direct bump, nothing independently notable.
- `ko` is used as a blank import in `hack/tools/tools.go:43` and built as a standalone binary by `hack/tools/prowimagebuilder/main.go:74` (`go build -o _bin/ko github.com/google/ko`), used only in the image-build pipeline.

## Dependency analysis

- **Freshness**: `v0.19.1` released 2026-06-29 (proxy.golang.org / GitHub release). ~2 months old at review time — fine, well past soak time.
- **Usage**: direct dependency of `hack/tools` only, a separate go.mod for build tooling. Not linked into any Prow runtime component. Exposure limited to the image-build pipeline (CI/release tooling), not a sensitive runtime path.
- **Changelog (v0.18.1 → v0.19.1)**: v0.19.0 added a repeatable `--ldflags` flag, fixed `defaultFlags` in `.ko.yaml` being silently ignored, and fixed missing directory headers in kodata tar archives (some strict tar implementations failed extraction) — plus routine dep bumps. v0.19.1 added one fix: skip YAML encoding when a selector filters out all documents. No CVEs/security advisories in range.
- **Exposure verdict**: light. Build-time tool only, no change to how we invoke it.
- **Take**: safe to take, already merged with no reported issues.

## Findings

None.

## Checked

- Confirmed no project source files (only `hack/tools/go.mod`/`go.sum`) changed between base `995ff1ae3` and head `00ca0969b`.
- Confirmed `ko` import surface: `hack/tools/tools.go:43` (blank import) and `hack/tools/prowimagebuilder/main.go:74` (built as `_bin/ko`).
- Confirmed release age and changelog contents for v0.19.0/v0.19.1 via GitHub releases and commit compare.

## Open questions

None.

## Dependency followups

Examined at 2026-10-22T16:16:19Z. PR #902 merged at 2026-08-26T06:19:39Z as `9a587d94024f159171ab0d71c8482cf2b064bc16`. One opportunity accepted; none skipped.

### [accepted] Centralize linker flags in prowimagebuilder

- Category: simplification. Necessity: could — optional maintenance cleanup.
- Unlocking dependency: `github.com/google/ko` `v0.18.1 → v0.19.1`; repeatable `--ldflags` added in v0.19.0.
- Release note: "Add repeatable --ldflags flag for go build" ([v0.19.0](https://github.com/ko-build/ko/releases/tag/v0.19.0)). The [implementation and documentation](https://github.com/ko-build/ko/pull/1680) establish that CLI linker flags override the complete YAML linker configuration.
- Where: `.ko.yaml:55` (35 repetitive build entries), `hack/tools/prowimagebuilder/main.go:247` (per-image publish arguments), and `hack/tools/prowimagebuilder/main_test.go:416` (mocked ko command capture).
- Benefit: derive the component name from the already-known image directory and define common linker flags once instead of maintaining a separate build entry for each component.
- Tradeoff accepted: removing those YAML entries changes direct ko invocations; the followup must document explicit linker flags for users who need the current embedded metadata.
- Decision: accepted by the user in this session. Mode: deliver. Implementation is delegated by the following prompt; no implementation performed here.

```text
In kubernetes-sigs/prow, now that merged PR #902 — "chore(deps): bump github.com/google/ko from 0.18.1 to 0.19.1 in /hack/tools" — upgraded github.com/google/ko v0.18.1 → v0.19.1, simplify the image builder's linker configuration. Work from the current default branch containing merge commit 9a587d94024f159171ab0d71c8482cf2b064bc16.

ko v0.19.0 added a repeatable --ldflags option. Its documentation states: "When set, --ldflags takes precedence over ldflags in .ko.yaml" (https://github.com/ko-build/ko/pull/1680). The current .ko.yaml:55 builds section has 35 entries repeating -s -w, -X sigs.k8s.io/prow/pkg/version.Version={{.Env.VERSION}}, and -X sigs.k8s.io/prow/pkg/version.Name=<component>. Every configured component name equals the basename of its main path. hack/tools/prowimagebuilder/main.go:247 constructs one ko publish command per image and can derive that name from imageDef.Dir.

Task: construct the complete repeated --ldflags arguments in prowimagebuilder's buildAndPush path, deriving the name from the image directory and preserving the existing VERSION value. ko's CLI flags replace all configured ldflags, so supply stripping, version, and name together. Remove the redundant per-image builds entries from .ko.yaml while retaining all baseImageOverrides. Update the existing command assertions in hack/tools/prowimagebuilder/main_test.go and document the supported build entry point plus the explicit flags needed by direct ko invocations to retain version/name metadata. Avoid global environment mutation per image because image builds run concurrently.

Acceptance criteria:
- Each of the 35 previously configured components receives the same stripping options and embedded Version/Name values as before, including external plugins and builds for multiple platforms.
- The 35 duplicate build entries are removed, base image selection is preserved, and registry, tag, platform, tarball, and push behavior is preserved.
- Explicitly account for cmd/webhook-server, which is in .prow-images.yaml but currently has no builds entry; document any resulting change in stripping or metadata rather than assuming it previously had those flags.
- Existing mocked command assertions cover the complete linker arguments for representative regular and external-plugin images. Run the focused image-builder tests using the repository's supported tooling.
- Build documentation explains the effect on direct ko users and shows how to retain their metadata without the removed YAML entries.

Scope: limit changes to linker flag construction, removal of duplicate YAML build entries, focused existing tests, and relevant build documentation. Do not bump dependencies, change base images or publishing policy, or refactor unrelated image-building logic. Do not publish changes without an explicit request.
```

### Scope and evidence

- The local HEAD contains a saved-review commit on top of the PR head. `git merge-base HEAD upstream/main` equals the already-merged PR head, so the dependency inventory uses GitHub's recorded PR base `995ff1ae3758f326568553c5f1684e11b460c241` and head `00ca0969b4245e14b873439e4c8cb4513542b8cb`, avoiding an empty post-merge comparison.
- Reviewed the complete ko v0.18.1 → v0.19.1 commit comparison and both intermediate release notes (v0.19.0 and v0.19.1). Prow already uses YAML linker flags; the new CLI option is actionable because its image builder already knows the component path.
- The defaultFlags fix changes build flags, not linker flags, and `.ko.yaml` has no defaultFlags to migrate. Prow invokes `ko publish`, so the YAML-selector fix does not expose a relevant call site. There are no tracked symlinks or production GOFLAGS/toolexec overrides indicating a regression from the kodata boundary or GOFLAGS changes; Deck stages kodata with rsync in `cmd/deck/gather-static.sh:34`.
- Cross-referenced every changed manifest/checksum module against non-vendor Go imports. The only affected-module import in the tools module is the ko tracking import (`hack/tools/tools.go:43`); its binary is built at `hack/tools/prowimagebuilder/main.go:74`. Other changed transitive modules have no owned call sites under the bumped module.
- Matching imports outside hack/tools belong to the unchanged root module: fsnotify, googleapis/api annotations, google.golang.org/api, the parent genproto module, client-go, and go-test/deep. Root versions remain independently pinned (for example fsnotify v1.10.1, google.golang.org/api v0.293.0, client-go v0.36.4). Their tools-module changes do not unlock root-code API migrations. Pure transitive changes were screened for applicability rather than treated as independently reviewed changelogs.
- No additional dependency followups identified. No tests or image builds were run for this handoff-only task.

### Dependency inventory

Go tools module: 49 manifest changes (38 version changes, 5 additions, 6 removals), plus 43 checksum-only changes. The Go directive changes `1.26.0 → 1.26.3`. Additions/removals include module-path migrations such as cosign v2 → v3 and Docker → Moby.

| Module | Old | New | Role in hack/tools |
|---|---|---|---|
| `github.com/aws/aws-sdk-go-v2/service/ecr` | `v1.54.4` | `v1.56.0` | indirect |
| `github.com/aws/aws-sdk-go-v2/service/ecrpublic` | `v1.38.8` | `v1.38.11` | indirect |
| `github.com/awslabs/amazon-ecr-credential-helper/ecr-login` | `v0.11.0` | `v0.12.0` | indirect |
| `github.com/containerd/stargz-snapshotter/estargz` | `v0.18.1` | `—` | indirect |
| `github.com/coreos/go-oidc/v3` | `v3.17.0` | `v3.18.0` | indirect |
| `github.com/docker/cli` | `v29.2.0+incompatible` | `v29.5.3+incompatible` | indirect |
| `github.com/docker/distribution` | `v2.8.3+incompatible` | `—` | indirect |
| `github.com/docker/docker` | `v28.5.2+incompatible` | `—` | indirect |
| `github.com/docker/docker-credential-helpers` | `v0.9.4` | `v0.9.5` | indirect |
| `github.com/docker/go-connections` | `v0.6.0` | `v0.7.0` | indirect |
| `github.com/dprotaso/go-yit` | `v0.0.0-20250513223454-5ece0c5aa76c` | `v0.0.0-20260209000607-dfb86291624d` | indirect |
| `github.com/fsnotify/fsnotify` | `v1.10.0` | `v1.10.1` | indirect |
| `github.com/go-openapi/analysis` | `v0.25.0` | `v0.25.2` | indirect |
| `github.com/go-openapi/jsonreference` | `v0.21.5` | `v0.21.6` | indirect |
| `github.com/go-openapi/runtime` | `v0.31.0` | `v0.32.3` | indirect |
| `github.com/go-openapi/spec` | `v0.22.4` | `v0.22.5` | indirect |
| `github.com/go-openapi/strfmt` | `v0.26.2` | `v0.26.3` | indirect |
| `github.com/go-openapi/swag/conv` | `v0.26.0` | `v0.26.1` | indirect |
| `github.com/go-openapi/swag/typeutils` | `v0.26.0` | `v0.26.1` | indirect |
| `github.com/go-openapi/validate` | `v0.25.2` | `v0.25.3` | indirect |
| `github.com/google/certificate-transparency-go` | `v1.3.2` | `v1.3.3` | indirect |
| `github.com/google/go-containerregistry` | `v0.20.7` | `v0.21.7` | indirect |
| `github.com/google/ko` | `v0.18.1` | `v0.19.1` | direct |
| `github.com/grpc-ecosystem/grpc-gateway/v2` | `v2.27.7` | `v2.29.0` | indirect |
| `github.com/in-toto/attestation` | `v1.1.2` | `v1.2.0` | indirect |
| `github.com/moby/moby/api` | `—` | `v1.55.0` | indirect |
| `github.com/moby/moby/client` | `—` | `v0.5.0` | indirect |
| `github.com/secure-systems-lab/go-securesystemslib` | `v0.10.0` | `v0.11.0` | indirect |
| `github.com/sigstore/cosign/v2` | `v2.6.2` | `—` | indirect |
| `github.com/sigstore/cosign/v3` | `—` | `v3.1.1` | indirect |
| `github.com/sigstore/protobuf-specs` | `v0.5.0` | `v0.5.1` | indirect |
| `github.com/sigstore/rekor` | `v1.5.0` | `v1.5.2` | indirect |
| `github.com/sigstore/rekor-tiles/v2` | `v2.0.1` | `v2.2.2-0.20260601073857-5d098a2b6443` | indirect |
| `github.com/sigstore/sigstore` | `v1.10.6` | `v1.10.8` | indirect |
| `github.com/sigstore/sigstore-go` | `v1.1.4` | `v1.2.0` | indirect |
| `github.com/sigstore/timestamp-authority/v2` | `v2.1.0` | `v2.1.2` | indirect |
| `github.com/theupdateframework/go-tuf/v2` | `v2.4.1` | `v2.4.2` | indirect |
| `github.com/transparency-dev/formats` | `v0.0.0-20251208091212-1378f9e1b1b7` | `v0.1.1` | indirect |
| `github.com/vbatts/tar-split` | `v0.12.2` | `—` | indirect |
| `github.com/youmark/pkcs8` | `—` | `v0.0.0-20240726163527-a2c0da244d78` | indirect |
| `go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp` | `v0.67.0` | `v0.68.0` | indirect |
| `go.opentelemetry.io/otel` | `v1.43.0` | `v1.44.0` | indirect |
| `go.opentelemetry.io/otel/metric` | `v1.43.0` | `v1.44.0` | indirect |
| `go.opentelemetry.io/otel/trace` | `v1.43.0` | `v1.44.0` | indirect |
| `go.uber.org/automaxprocs` | `v1.6.0` | `—` | indirect |
| `go.yaml.in/yaml/v4` | `—` | `v4.0.0-rc.6` | indirect |
| `google.golang.org/genproto/googleapis/api` | `v0.0.0-20260414002931-afd174a4e478` | `v0.0.0-20260526163538-3dc84a4a5aaa` | indirect |
| `google.golang.org/genproto/googleapis/rpc` | `v0.0.0-20260427160629-7cedc36a6bc4` | `v0.0.0-20260523011958-0a33c5d7ca68` | indirect |
| `sigs.k8s.io/kind` | `v0.30.0` | `v0.32.0` | indirect |

Checksum-only entries below are a checksum inventory, not evidence of the selected build-list version.

| Module | Old checksum versions | New checksum versions |
|---|---|---|
| `cloud.google.com/go/iam` | `v1.7.0` | `v1.11.0` |
| `cloud.google.com/go/longrunning` | `v0.9.0` | `v1.0.0` |
| `filippo.io/mldsa` | `—` | `v0.0.0-20260215214346-43d0283efc3e` |
| `github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azkeys` | `v1.4.0` | `v1.5.0` |
| `github.com/AzureAD/microsoft-authentication-library-for-go` | `v1.6.0` | `v1.7.2` |
| `github.com/aws/aws-sdk-go` | `v1.55.8` | `—` |
| `github.com/aws/aws-sdk-go-v2/service/kms` | `v1.51.1` | `v1.52.0` |
| `github.com/containerd/log` | `v0.1.0` | `—` |
| `github.com/creack/pty` | `v1.1.18` | `v1.1.24` |
| `github.com/go-chi/chi` | `v4.1.2+incompatible` | `—` |
| `github.com/go-chi/chi/v5` | `v5.2.5` | `v5.3.0` |
| `github.com/go-openapi/testify/enable/yaml/v2` | `v2.5.0` | `v2.5.1` |
| `github.com/go-openapi/testify/v2` | `v2.5.0` | `v2.5.1` |
| `github.com/go-sql-driver/mysql` | `v1.9.3` | `—` |
| `github.com/go-test/deep` | `—` | `v1.1.1` |
| `github.com/google/pprof` | `v0.0.0-20260115054156-294ebfa9ad83` | `v0.0.0-20260402051712-545e8a4df936` |
| `github.com/google/trillian` | `v1.7.2` | `v1.7.3` |
| `github.com/googleapis/enterprise-certificate-proxy` | `v0.3.15` | `v0.3.16` |
| `github.com/jackc/pgpassfile` | `v1.0.0` | `—` |
| `github.com/jackc/pgservicefile` | `v0.0.0-20240606120523-5a60cdf6a761` | `—` |
| `github.com/jackc/pgx/v5` | `v5.7.5` | `—` |
| `github.com/jackc/puddle/v2` | `v2.2.2` | `—` |
| `github.com/jmespath/go-jmespath` | `v0.4.1-0.20220621161143-b0104c826a24` | `—` |
| `github.com/letsencrypt/boulder` | `v0.20260223.0` | `v0.20260309.0` |
| `github.com/moby/sys/atomicwriter` | `v0.1.0` | `—` |
| `github.com/moby/sys/sequential` | `v0.6.0` | `—` |
| `github.com/morikuni/aec` | `v1.0.0` | `—` |
| `github.com/prashantv/gostub` | `v1.1.0` | `—` |
| `github.com/sergi/go-diff` | `v1.3.1` | `v1.4.0` |
| `github.com/sigstore/sigstore/pkg/signature/kms/aws` | `v1.10.6` | `v1.10.8` |
| `github.com/sigstore/sigstore/pkg/signature/kms/azure` | `v1.10.6` | `v1.10.8` |
| `github.com/sigstore/sigstore/pkg/signature/kms/gcp` | `v1.10.6` | `v1.10.8` |
| `github.com/sigstore/sigstore/pkg/signature/kms/hashivault` | `v1.10.6` | `v1.10.8` |
| `github.com/tink-crypto/tink-go-awskms/v2` | `v2.1.0` | `—` |
| `go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc` | `v0.67.0` | `v0.68.0` |
| `go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp` | `v1.33.0` | `—` |
| `go.opentelemetry.io/otel/sdk` | `v1.43.0` | `v1.44.0` |
| `go.opentelemetry.io/otel/sdk/metric` | `v1.43.0` | `v1.44.0` |
| `go.step.sm/crypto` | `v0.81.0` | `v0.77.7` |
| `google.golang.org/api` | `v0.277.0` | `v0.283.0` |
| `google.golang.org/genproto` | `v0.0.0-20260319201613-d00831a3d3e7` | `v0.0.0-20260406210006-6f92a3bedf2d` |
| `k8s.io/client-go` | `v0.36.0` | `v0.36.1` |
| `pgregory.net/rapid` | `—` | `v1.2.0` |
