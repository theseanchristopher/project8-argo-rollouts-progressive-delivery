# Failure and Rollback

This document explains what happens when the canary analysis fails, how Argo Rollouts reacts, and what to inspect.

---

## 1. Failure Conditions in This Project

The canary gate fails when:

1. The Prometheus query returns a value > 0 (restarts occurred), or
2. The Prometheus query returns no data (`[]`) and we fail closed

This conservative approach is intentional. In production, “no data” should not equal “healthy.”

---

## 2. What Argo Rollouts Does on Failure

When analysis fails:

1. The AnalysisRun enters `Failed` (or `Error` if query evaluation fails)
2. The Rollout stops progressing through steps
3. Traffic is shifted back to the stable ReplicaSet (depending on routing configuration)
4. Canary ReplicaSet scales down
5. The rollout reports rollback to the previous stable revision

---

## 3. How to Confirm Rollback

### 3.1 Inspect rollout status
```bash
kubectl describe rollout project8-nginx-rollout -n project8-dev
```

Look for:
- current revision
- aborted/rollback messages
- step history

### 3.2 Inspect ReplicaSets
```bash
kubectl get rs -n project8-dev --sort-by=.metadata.creationTimestamp
```

You should see:
- stable ReplicaSet still running
- canary ReplicaSet scaled down or reduced

---

## 4. How to Inspect the Failed AnalysisRun

```bash
kubectl get analysisruns -n project8-dev --sort-by=.metadata.creationTimestamp
kubectl describe analysisrun -n project8-dev <NAME>
```

Look at:
- resolved query
- measurement values
- final message explaining the failure condition

---

## 5. Why Rollback Is a Feature, Not a Bug

The entire value proposition of progressive delivery is:
- you can safely attempt releases
- failures are expected and handled automatically
- rollback is fast and predictable

