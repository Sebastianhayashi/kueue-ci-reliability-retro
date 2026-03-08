# Async Visibility

Tests fail because they treat distributed async systems as immediately consistent.

## Pattern

Both cases share the same root cause: test code assumes an API-server write is instantly visible to all consumers (webhooks, cross-cluster controllers). It isn't. Propagation lag is variable and CI load makes it worse. This will keep producing flakes wherever a test creates a resource and immediately depends on its visibility elsewhere.

## Cases

### [#9571](https://github.com/kubernetes-sigs/kueue/pull/9571) — Webhook cache lag rejects TrainJob

`admission webhook "vtrainjob.kb.io" denied the request: runtime 'test-trainingruntime' not found`

Test creates `TrainingRuntime` then immediately creates `TrainJob` — webhook cache hasn't indexed the runtime yet.

```go
// test/e2e/tas/trainjob_test.go — BEFORE (3 occurrences)
util.MustCreate(ctx, k8sClient, trainingRuntime)
util.MustCreate(ctx, k8sClient, trainjob)   // webhook cache miss → rejected
```

```diff
// FIX: retry-aware creation tolerates cache lag
-  util.MustCreate(ctx, k8sClient, trainjob)
+  createTrainJobWithRetry(ctx, k8sClient, trainjob)

+func createTrainJobWithRetry(ctx context.Context, c client.Client, trainJob *kftrainer.TrainJob) {
+    gomega.Eventually(func(g gomega.Gomega) {
+        err := c.Create(ctx, trainJob)
+        g.Expect(client.IgnoreAlreadyExists(err)).ToNot(gomega.HaveOccurred())
+    }, util.Timeout, util.Interval).Should(gomega.Succeed())
+}
```

[Failing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_kueue/9501/pull-kueue-test-e2e-tas-release-0-16/2026834045899378688) · [Passing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_kueue/9625/pull-kueue-test-e2e-tas-release-0-16/2028425299724603392)

### [#9572](https://github.com/kubernetes-sigs/kueue/pull/9572) — Cross-cluster completion exceeds single-cluster timeout

`[FAILED] Timed out after 45.001s.`

MultiKueue TAS workload completes on worker but `WorkloadFinished` doesn't propagate to manager within `LongTimeout`. The signal must traverse worker → adapter → manager — a multi-hop path.

```go
// test/e2e/multikueue/tas_test.go — BEFORE (2 occurrences)
}, util.LongTimeout, util.Interval).Should(gomega.Succeed())
//   ^^^^^^^^^^^^^^ single-cluster budget on a multi-hop path
```

```diff
-}, util.LongTimeout, util.Interval).Should(gomega.Succeed())
+}, util.VeryLongTimeout, util.Interval).Should(gomega.Succeed())
```

[Failing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/logs/periodic-kueue-test-e2e-multikueue-release-0-16/2026170270561079296) · [Passing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_kueue/9584/pull-kueue-test-e2e-multikueue-release-0-16/2027423858969022464)

## Related issues

- [#9301](https://github.com/kubernetes-sigs/kueue/issues/9301) (open) — conversion webhook EOF during setup
- [#9523](https://github.com/kubernetes-sigs/kueue/issues/9523) (open) — 300s insufficient for KubeRay worker visibility
- [#9609](https://github.com/kubernetes-sigs/kueue/issues/9609) (closed) — historical timing flake

## Fix direction

**How to recognize this family**: the test fails with a timeout or "not found" error on a resource that was just created in a preceding step.

1. **Visibility-wait before dependent creates** — use `Eventually` to confirm resource is readable by consumers before proceeding.
2. **Timeout tiers** — multi-hop / cross-cluster paths should use `VeryLongTimeout` by default, not share single-cluster budgets.

## Scope

This family covers test-side timing assumptions only. Production controller reconciliation latency is out of scope.
