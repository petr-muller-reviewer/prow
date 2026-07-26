---
issue: kubernetes-sigs/prow#603
title: "flaking test: `TestAddWithParser`"
state: open
labels: kind/bug, lifecycle/rotten
main_sha: 71428b9c282ee8c9e7e9512068fccce86e7915da
triaged_at: 2026-06-01T10:55:35Z
verdict: accepted
refresh_log:
  - previous: 2026-05-30T23:20:59Z
    summary: "lifecycle/stale progressed to lifecycle/rotten (automated bot). No new comments, no cross-references, no linked PRs. Main advanced 4 commits, none touching pkg/config/secret/."
---

## Findings

### [cause] Race between parser channel send and lock acquisition
- detail: The test's `parsingFN` sends parsed value to a channel inside the parser callback. The parser executes inside `loadSingleSecretWithParser` (reloader.go:72), before `reloadSecret` acquires the write lock and commits `p.parsed` (reloader.go:78-80). The test receives on the channel then immediately calls `generator()`, which can read the stale value.
- evidence: `pkg/config/secret/reloader.go:72-81`, `pkg/config/secret/agent_test.go:203`, `pkg/config/secret/agent_test.go:227`

### [cause] Production code is correct
- detail: The `RWMutex` on `parsingSecretReloader` properly synchronizes reads (`get()` with `RLock`) and writes (reload with `Lock`). The race is entirely in the test's synchronization strategy.
- evidence: `pkg/config/secret/reloader.go:78-81`, `pkg/config/secret/reloader.go:95-99`

### [reproducibility] Observed twice on unrelated PRs
- detail: Flake appeared on PR #601 and PR #525 in CI (`pull-prow-unit-test`). Both failures show same pattern: `expected value 2 from generator, got 1`.
- evidence: https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_prow/601/pull-prow-unit-test/2015379343022231552, https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_prow/525/pull-prow-unit-test/2015813939778031616

### [related-code] Reload loop — parser runs before lock
- where: `pkg/config/secret/reloader.go:50-87`
- excerpt: |
    raw, parsed, err := loadSingleSecretWithParser(p.path, p.parsingFN) // line 72
    if err != nil {
        logger.WithField("secret-path", p.path).WithError(err).Error("Error loading secret.")
        continue
    }
    p.lock.Lock()       // line 78
    p.rawValue = raw
    p.parsed = parsed   // line 80
    p.lock.Unlock()

### [related-code] Test's racy checkValueAndErr
- where: `pkg/config/secret/agent_test.go:211-229`
- excerpt: |
    select {
    case v := <-vals:
        if v != expected {
            t.Errorf("expected value to get updated to %d but got updated to %d", expected, v)
        }
    case err := <-errs:
        ...
    case <-time.After(10 * time.Second):
        ...
    }
    if actual := generator(); actual != expected {  // line 227 — races with lock
        t.Errorf("expected value %d from generator, got %d", expected, actual)
    }

### [related-code] Parser callback sends to channel before lock
- where: `pkg/config/secret/agent_test.go:197-205`
- excerpt: |
    func(raw []byte) (int, error) {
        val, err := strconv.Atoi(string(raw))
        if err != nil {
            errs <- err
            return val, err
        }
        vals <- val     // channel send happens inside parser, before lock acquisition
        return val, err
    }

### [related-code] AddWithParser returns loader.get as generator
- where: `pkg/config/secret/agent.go:75-81`
- excerpt: |
    func AddWithParser[T any](path string, parsingFN func([]byte) (T, error)) (func() T, error) {
        loader := &parsingSecretReloader[T]{
            path:      path,
            parsingFN: parsingFN,
        }
        return loader.get, secretAgent.add(path, loader)
    }

### [related-code] Singleton secretAgent shared by parallel test instances
- where: `pkg/config/secret/agent.go:33`
- excerpt: |
    var secretAgent *agent

## Checked
- Full read of `pkg/config/secret/`: agent.go, reloader.go, secret.go, agent_test.go
- Synchronization primitives: `sync.RWMutex` on `parsingSecretReloader` and `agent`, no atomics
- Whether production code has a bug: no, `RWMutex` correctly guards `p.parsed`
- Whether singleton `secretAgent` causes cross-test interference: adds scheduling pressure, not root cause
- Reload interval: 1 second (`time.Tick(1 * time.Second)` at reloader.go:55)
- Available area labels: none matching `pkg/config/secret`

## Next steps
- Post comment with root cause and fix approach
- Apply labels: `/remove-lifecycle rotten`, `/kind flake`, `/good-first-issue`
- Fix is test-only: replace channel-based sync in `checkValueAndErr` with polling `generator()` + timeout
- Verify fix with `go test -race -count=100 ./pkg/config/secret/`

## Open questions
- Is this still flaking on current main, or has it gone quiet since January 2026?
- Should the `skips` optimization in `reloadSecret` (reloader.go:54-70) also get a test, or is that out of scope?
