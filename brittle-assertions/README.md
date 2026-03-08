# Brittle Assertions

Tests fail because assertions or performance baselines are too tight for real CI variance.

**Logic**: CI retry counts and scheduling latency fluctuate naturally → overly strict bounds convert variance into failures.

---

## [#9554](https://github.com/kubernetes-sigs/kueue/pull/9554) — Admission-attempt upper bound too tight

Scheduler integration test asserts a workload is processed at most **2 times** before settling as inadmissible. Under CI load a CQ-creation requeue can trigger a third attempt.

```go
// test/integration/singlecluster/scheduler/scheduler_test.go — BEFORE
util.ExpectPendingAdmissionAttempts(2, "<=")
```

```diff
-  util.ExpectPendingAdmissionAttempts(2, "<=")
+  util.ExpectPendingAdmissionAttempts(3, "<=")
```

Two occurrences in the same test (pre- and post-CQ-update assertions).

[Failing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/logs/periodic-kueue-test-integration-release-0-15/2027016273371598848) · [Passing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_kueue/9564/pull-kueue-test-integration-baseline-release-0-15/2027409767902744576)

---

## [#9581](https://github.com/kubernetes-sigs/kueue/pull/9581) — Performance baseline threshold too narrow

`Average wait ... more then expected`

Scheduling performance test uses a `+5% + 6s` margin over the measured average for the `small` workload class. Real variance exceeds this.

```yaml
# test/performance/scheduler/configs/baseline/rangespec.yaml — BEFORE
# Average value 215468 (+/- 2%), setting at +5% + 6s accounting for observed failures
small: 233_000
```

```diff
-  # Average value 215468 (+/- 2%), setting at +5% + 6s accounting for observed failures
-  small: 233_000
+  # Average value 215468 (+/- 2%), setting at +20% accounting for observed failures
+  small: 260_000
```

[Failing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_kueue/9340/pull-kueue-test-scheduling-perf-main/2024397273403756544) · [Passing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_kueue/9585/pull-kueue-test-scheduling-perf-release-0-16/2027417034228240384)

---

## Recommendations

1. **Model expected variance explicitly** — bounds should include a documented margin (e.g. +20%) derived from observed CI behavior, not theoretical ideals.
2. **Treat retry counts as ranges** — Kubernetes controllers can reprocess objects due to informer resyncs or watch restarts; assertions should use `>=`/`<=` ranges rather than exact counts.
