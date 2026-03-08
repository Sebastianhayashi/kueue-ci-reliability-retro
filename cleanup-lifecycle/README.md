# Cleanup Lifecycle

Tests fail because cleanup routines are not idempotent — they treat already-completed states as errors.

**Logic**: Cleanup can run after partial or full teardown → hitting a terminal state (e.g. cluster already deleted) should be success, not failure.

---

## [#9577](https://github.com/kubernetes-sigs/kueue/pull/9577) — `kind delete cluster` fails on already-deleted cluster

`ERROR: unknown cluster "kind"`

E2E cleanup retries `kind delete cluster`, but if the cluster was already removed (race, previous partial cleanup), the command returns a non-zero exit code with `unknown cluster`.

```bash
# hack/testing/e2e-common.sh — BEFORE
for attempt in $(seq 1 "$max_retries"); do
    if $KIND delete cluster --name "$cluster_name"; then
        return 0
    fi
    # ... retry
```

The original code retries on failure but never checks **why** it failed — an already-deleted cluster is not a transient error, it's the desired end state.

```diff
     for attempt in $(seq 1 "$max_retries"); do
-        if $KIND delete cluster --name "$cluster_name"; then
+        local output
+        if output=$($KIND delete cluster --name "$cluster_name" 2>&1); then
+            echo "$output"
+            return 0
+        fi
+        echo "$output"
+
+        if [[ "$output" == *"unknown cluster"* ]]; then
+            echo "Cluster '$cluster_name' is already deleted (unknown cluster)."
             return 0
         fi
```

[Failing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/logs/periodic-kueue-test-e2e-main-1-34/2024217633649332224) · [Passing run](https://prow.k8s.io/view/gs/kubernetes-ci-logs/pr-logs/pull/kubernetes-sigs_kueue/9583/pull-kueue-test-e2e-release-0-16-1-34/2027406963704336384)

---

## Systematic evidence

- [#9582](https://github.com/kubernetes-sigs/kueue/issues/9582) (open) — populator cluster deletion intermittently fails
- [#9597](https://github.com/kubernetes-sigs/kueue/pull/9597) (open, WIP) — cleanup/helper reuse convergence

## Recommendations

1. **Idempotent cleanup** — cleanup functions should treat "already gone" as success. Check error output for terminal-state indicators before retrying.
2. **Guard shared teardown** — when multiple test suites share teardown helpers, each invocation should be safe to run regardless of prior state.
