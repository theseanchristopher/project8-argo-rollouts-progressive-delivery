# Failure and Rollback

This document explains what happens when canary analysis fails, how Argo Rollouts reacts, and what to inspect to confirm rollback.

---

## 1. Failure Conditions in Project 8

The canary gate fails when:

1. The Prometheus query returns a value **greater than 0** (restarts occurred), or
2. The Prometheus query returns **no data** (`[]`) and the gate fails closed

This conservative approach is intentional: in production, missing telemetry is not a valid signal of health.

---

## 2. What Argo Rollouts Does on Failure

When analysis fails:

1. The AnalysisRun transitions to `Failed` (or `Error` for provider/query problems).
2. The Rollout stops progressing through steps.
3. The rollout aborts the canary attempt.
4. The stable revision remains (or becomes) the active revision.
5. Canary ReplicaSet scales down (depending on strategy settings and traffic routing).

The key outcome: the system does not continue promoting a revision that failed its gate.

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

- the stable ReplicaSet running
- the canary ReplicaSet scaled down or reduced

---

## 4. Inspect the Failed AnalysisRun

```bash
kubectl get analysisruns -n project8-dev --sort-by=.metadata.creationTimestamp
kubectl describe analysisrun -n project8-dev <NAME>
```

Look at:

- resolved query
- measurement values
- final message explaining which condition triggered the failure

---

## 5. Why Rollback Is a Feature

Progressive delivery assumes failures will happen:

- you attempt releases safely
- the system detects problems quickly
- rollback is automatic and repeatable
- the failure is recorded for audit and debugging

This is one of the clearest real-world benefits of Argo Rollouts compared to a standard Deployment rollout.
