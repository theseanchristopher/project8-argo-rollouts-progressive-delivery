# Rollout Execution (How to Run and Verify the Canary)

This document explains how to validate the rollout is configured correctly and how to observe canary execution, including AnalysisRuns.

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

### 1.3 Verify analysis step is present in the applied Rollout
```bash
kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml | grep -n "analysis:"
```

If this returns nothing, your overlay patch likely didn’t apply (see `docs/troubleshooting.md`).

---

## 2. Triggering a New Revision (Critical)

Argo Rollouts executes canary steps (including analysis) when a **new revision** is created.

A new revision occurs when the Rollout **pod template changes**, commonly:
- image tag changes (most common)
- pod template annotations change
- env vars / container args change

If you “release” without changing the pod template, the revision will not increment and you may not get a new AnalysisRun.

### How this project triggers revisions
In the broader portfolio flow:
- Project 1’s CI pushes a new image tag
- GitOps updates the tag (or manifests) so Argo CD sync triggers a new ReplicaSet

---

## 3. Watch the Rollout

If you have the `kubectl argo rollouts` plugin:

```bash
kubectl argo rollouts get rollout project8-nginx-rollout -n project8-dev --watch
```

What to look for:
- revision number increment
- step progression (setWeight, pause, analysis)
- whether analysis is running / passed / failed

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

Important fields:
- **Resolved Prometheus Query** (what was actually executed)
- Measurements (each evaluation)
- Final Phase (Successful / Failed / Error)
- Message (most useful human-readable failure reason)

---

## 5. Verify Prometheus Result Matches the Gate

Port-forward Prometheus and run the same query directly:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
curl -s "http://localhost:9090/api/v1/query?query=sum(increase(kube_pod_container_status_restarts_total%7Bnamespace%3D%22project8-dev%22%7D%5B2m%5D))"
```

Expected:
- a numeric value, often `0`

If you see `[]`:
- the metric does not exist, or
- your selector matches nothing (wrong namespace), or
- kube-state-metrics is missing/un-scraped

---

## 6. What “Success” Looks Like

- Rollout advances beyond the analysis step
- AnalysisRun shows a consistent `0` value
- Rollout continues to the next canary weight / pause step

---

## 7. What “Failure” Looks Like

- AnalysisRun phase becomes Failed (or Error)
- Rollout aborts and shifts back to the stable revision
- Canary ReplicaSet scales down
- Rollout indicates rollback to a prior revision

Details: `docs/failure-and-rollback.md`
