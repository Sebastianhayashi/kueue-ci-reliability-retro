# Kueue CI Reliability

Recurring flaky-test patterns in [kubernetes-sigs/kueue](https://github.com/kubernetes-sigs/kueue), grouped by root cause with minimal evidence.

| Category | Core problem | PRs |
|---|---|---|
| [`async-visibility`](async-visibility/) | Treats distributed async writes as immediately visible | [#9571](https://github.com/kubernetes-sigs/kueue/pull/9571), [#9572](https://github.com/kubernetes-sigs/kueue/pull/9572) |
| [`brittle-assertions`](brittle-assertions/) | Converts normal CI variance into test failures | [#9554](https://github.com/kubernetes-sigs/kueue/pull/9554), [#9581](https://github.com/kubernetes-sigs/kueue/pull/9581) |
| [`cleanup-lifecycle`](cleanup-lifecycle/) | Cleanup paths lack idempotent terminal-state handling | [#9577](https://github.com/kubernetes-sigs/kueue/pull/9577) |

