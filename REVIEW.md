---
pr: kubernetes-sigs/prow#1026
title: "Require rationale for /retest, /retest-required, and /override"
head_sha: 1e4f822088659f761c582b695d1fae19c87f546d
base: main
reviewed_at: 2026-10-29T19:10:45Z
verdict: request-changes
gate:
  decision: do-not-merge
  gated_at: 2026-10-28T01:14:05Z
  gated_head_sha: 1e4f822088659f761c582b695d1fae19c87f546d
  reviewed_head_sha: 1e4f822088659f761c582b695d1fae19c87f546d
refresh_log:
  - at: 2026-10-29T19:10:45Z
    old_sha: 1e4f822088659f761c582b695d1fae19c87f546d
    new_sha: 1e4f822088659f761c582b695d1fae19c87f546d
    summary: "No code changes; only activity since the review was BenTheElder's `/assign @petr-muller` comment. Findings unchanged, blocking override bypass still open."
---

# Review

## Gate

**Decision: do-not-merge** at `1e4f822088659f761c582b695d1fae19c87f546d`.

The current PR head is identical to the reviewed head: no subsequent commits address the confirmed override rationale bypass. A failed initial authorization lookup still skips validation, while a successful authorization lookup in the native handler can permit an override without the required rationale. This remains an unresolved blocking finding; fix the sequencing and cover the failed-then-successful lookup scenario before merging. GitHub contains no submitted reviews or inline review comments adding further gating findings; the author's validation evidence does not cover this failure transition.

### Findings disposition

- **Not addressed; blocks merge** — source: local `REVIEW.md`, authorization/validation sequencing at `pkg/plugins/override/override.go:343-344`. The current code skips validation on a false preliminary result, continues through line 371, and checks permission again at line 533. Permission lookup errors still return false at lines 384-388.
- **Not addressed; accepted as nonblocking** — source: local `REVIEW.md`, optional parser compatibility tests at `pkg/plugins/rationale.go:51-53`. Command recognition remains duplicated, but no concrete divergence was identified; this suggestion does not drive the gate decision.

### Gating list

1. Fix the unresolved override rationale bypass at `pkg/plugins/override/override.go:343-344` (source: local blocking finding), with a regression test for the first authorization lookup failing and a later lookup succeeding.

### Independent merge risk

- **API/configuration:** inspection found additive exported types, functions, and struct fields, plus optional `require_rationale` configuration; no removed or renamed public API, CRD changes, or changed defaults for existing configurations. No notable compatibility risk for deployments without the policy configured.
- **Behavior:** repositories opting in can have all trigger or override actions in a combined comment suppressed by invalid rationale; this is documented. Protected override requests also repeat PR/authorization lookups, adding API work only for configured override traffic. Neither issue adds an independent merge blocker.
- **Rollout:** update command-posting automation and finish upgrading hook replicas before relying on enforcement; older binaries do not implement the policy. Removing the policy restores prior command behavior without a stored-state migration.
- This pass inspected the unchanged PR's API, configuration, and handler changes directly; no additional specialized compatibility skill applied.

## Verdict

Request changes for a rationale policy bypass in override handling.

All three maintainer perspectives identified the same confirmed defect: a failed preliminary override permission lookup skips rationale validation, while a successful second lookup can permit status or check-run mutations without the required rationale. Fix this sequencing and add a regression test before merging. The feature otherwise has a limited deployment footprint because configuration is opt-in; operators should update command-posting automation, complete the hook rollout before relying on enforcement, and initially enable the policy for a few repositories. Rejection also suppresses other commands handled by that plugin in the same comment, as documented.

## What this PR does

- Adds opt-in, exact-repository policies for rationale requirements on `/retest`, `/retest-required`, and `/override`.
- Parses same-line and following-line rationales, validates each command occurrence, and strips same-line rationale text before native command parsing.
- Blocks the trigger or override handler when an authorized user supplies an invalid rationale, and documents configuration and behavior.

Since previous review (2026-10-28T01:09:54Z, `1e4f8220`): no code changed.

- PR head is still `1e4f822088659f761c582b695d1fae19c87f546d`; `git diff 1e4f8220..FETCH_HEAD` for `pull/1026/head` is empty, so no commits were pushed and no force-push occurred.
- Only activity since the review: BenTheElder's `/assign @petr-muller` issue comment at 2026-10-28T01:29:35Z (with the resulting assignment/subscription timeline events). No submitted reviews, inline review comments, label changes, or discussion of the findings.
- The blocking override rationale bypass at `pkg/plugins/override/override.go:343-344` is untouched and still gates merge.

## Findings

### [blocking] Tie rationale validation to the authorization that permits execution
- where: `pkg/plugins/override/override.go:343-344`
- concern: CONFIRMED by code inspection and flagged by all three reviewers: for a protected repository, `/override job-a` skips rationale validation if the preliminary permission lookup fails, because `isAuthorized` treats lookup errors as false. The native handler independently checks authorization again at line 533; if that lookup succeeds, it can update statuses or check runs without a rationale. Validate after the authorization check that permits execution, or stop processing when the preliminary check fails, and add a regression test covering a failed initial lookup followed by a successful lookup.
- excerpt: |
    if isAuthorized(oc, logger, org, repo, e.User.Login, options, pr) {
    	if err := plugins.ValidateRationale(e.Body, "/override", settings); err != nil {

### [should-fix] Consider tests that detect command grammar divergence
- where: `pkg/plugins/rationale.go:51-53`
- concern: Optional maintainability suggestion: rationale command recognition duplicates the native override expression and explicitly requires keeping the two synchronized. Consider sharing the grammar or adding compatibility tests so a future command syntax change cannot silently diverge from rationale enforcement; this suggestion does not block merge.
- excerpt: |
    "override": {
    	// Keep this syntax in sync with override.overrideRe.
    	matcher:   regexp.MustCompile(`(?i)^/override(?: ([^\r\n]+))?[\r\n]?$`),

## Resolved

### [question] Withdrawn: honor or reject an explicit zero minimum
- where: `pkg/plugins/config.go:173-178`
- concern: Withdrawn after the defect-focused review. The implementation and tests consistently use zero to select the documented default; the earlier finding did not establish an unintended behavior or concrete regression.
- excerpt: |
    func (s RationaleSettings) EffectiveMinLength() int {
    	if s.MinLength == 0 {
    		return defaultRationaleMinLength
    	}
    	return s.MinLength
    }

## Checked

- Trigger rationale validation follows trust checks and returns before job creation when rationale is invalid.
- Override validation blocks ordinary authorized-user requests with invalid rationale before mutations; the failed-then-successful authorization path is the exception reported above.
- Multiple command occurrences, following-line boundaries, code fences, and same-line argument stripping are handled consistently with the documented behavior.
- `CONTRIBUTING.md` contains no repo-specific coding rules; no clear baseline code smell was found.
- Passed: `go test ./pkg/plugins ./pkg/plugins/override ./pkg/plugins/trigger -run 'Rationale|HandleGenericComment' -count=1`.
- Existing authorization tests use consistently authorized or unauthorized users; they do not cover a failed permission lookup followed by a successful lookup in the same comment handler.
- Existing configurations retain their behavior without the new policy; no new services, persistent state, Kubernetes RBAC, or GitHub permission requirements were introduced.
- Structured rejection logs include actor, repository, pull request, and command; documentation covers syntax, bot exemptions, and interactions with other commands.

## Open questions

None raised during the maintainer walkthrough.
