# Rollout Execution (How to Run and Verify the Canary)

This document explains how to validate that the rollout is configured correctly and how to observe canary execution, including AnalysisRuns.

---

## 1. Pre-Flight Checklist

### 1.1 Verify Rollout exists
```bash
kubectl get rollout -n project8-dev
```

### 1.2 Verify AnalysisTemplate exists
```bash
kubectl get analysistemplate -n project8-dev
```

### 1.3 Verify the applied Rollout includes an analysis step
```bash
kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml | grep -n "analysis:"
```

If this returns nothing, your dev overlay patch likely didn’t apply (see `docs/troubleshooting.md`).

---

## 2. Triggering a New Revision (Critical)

Argo Rollouts executes canary steps (including analysis) when a **new revision** is created.

A new revision occurs when the Rollout **pod template changes**, commonly:

- image tag changes (most common)
- pod template annotations change
- env vars / container args change

If you “release” without changing the pod template, the revision will not increment and you will not get a new AnalysisRun.

---

## 3. Watch the Rollout

If you have the `kubectl argo rollouts` plugin installed:

```bash
kubectl argo rollouts get rollout project8-nginx-rollout -n project8-dev --watch
```

What to look for:

- revision number increments
- step progression (setWeight → pause → analysis)
- analysis phase (Running → Successful/Failed)

---

## 4. Observe AnalysisRuns

### 4.1 List AnalysisRuns
```bash
kubectl get analysisruns -n project8-dev --sort-by=.metadata.creationTimestamp
```

### 4.2 Describe the latest AnalysisRun
```bash
kubectl describe analysisrun -n project8-dev <NAME>
```

Most useful fields:

- **Resolved query** (what was actually executed)
- measurement values (including `Value: []`)
- final phase and message

---

## 5. Validate the Gate Query Matches Prometheus

Port-forward Prometheus and run the same query:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
curl -s "http://localhost:9090/api/v1/query?query=sum(increase(kube_pod_container_status_restarts_total%7Bnamespace%3D%22project8-dev%22%7D%5B2m%5D))"
```

Expected:

- a numeric value (often `0`)

If you see `[]`:

- the metric doesn’t exist, or
- the selector matches nothing (wrong namespace), or
- kube-state-metrics isn’t installed/scraped

---

## 6. What Success Looks Like

- Rollout advances beyond the analysis step
- AnalysisRun shows consistent `0` values
- Rollout continues through remaining steps and promotes

---

## 7. What Failure Looks Like

- AnalysisRun phase becomes `Failed` (or `Error`)
- Rollout aborts and returns to the previous stable revision
- Canary ReplicaSet scales down

See `docs/failure-and-rollback.md`.
