# Installation and Prerequisites

This document captures what must be present for Project 8 to work end-to-end, and how to validate prerequisites before troubleshooting rollout behavior.

---

## 1. Prerequisites

### 1.1 Cluster access
```bash
kubectl cluster-info
kubectl get nodes
```

### 1.2 Namespaces
Project 8 deploys into environment namespaces:

- `project8-dev`
- `project8-pre`
- `project8-prod`

Verify (example: dev):

```bash
kubectl get ns project8-dev
```

### 1.3 Argo CD (GitOps)
Argo CD is assumed to be present for the GitOps workflow. If you are not using Argo CD, you can apply the same manifests manually using `kubectl apply -k`.

---

## 2. Install and Verify Argo Rollouts

### 2.1 What must be installed
Argo Rollouts requires:

- the rollouts controller deployment
- CRDs such as:
  - `rollouts.argoproj.io`
  - `analysistemplates.argoproj.io`
  - `analysisruns.argoproj.io`

### 2.2 Verify CRDs (critical)
```bash
kubectl get crd | grep argoproj.io | grep -E "rollouts|analysistemplates|analysisruns"
```

If these CRDs are missing, the resources used by this project cannot be created.

### 2.3 Verify controller pods
```bash
kubectl get pods -n argo-rollouts
```

You should see a running rollouts controller.

---

## 3. Prometheus Requirements

Project 8 assumes Prometheus is deployed (commonly via `kube-prometheus-stack` from Project 7).

### 3.1 Verify Prometheus Service exists
```bash
kubectl get svc -n monitoring | grep prometheus
```

### 3.2 Prometheus must be reachable from the rollouts controller
AnalysisRuns execute Prometheus queries *inside the cluster*. The AnalysisTemplate must point to an in-cluster service DNS name, for example:

- `http://kube-prometheus-stack-prometheus.monitoring:9090`

If Prometheus is unreachable, AnalysisRuns will fail with provider/connection errors.

---

## 4. Validate Prometheus Health and Metric Availability

### 4.1 Port-forward Prometheus (local validation)
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

### 4.2 Baseline query: `up`
```bash
curl -s "http://localhost:9090/api/v1/query?query=up" | head -c 400 && echo
```

### 4.3 Validate the gate metric exists
Project 8 uses the kube-state-metrics restart counter:

```bash
curl -s "http://localhost:9090/api/v1/query?query=count(kube_pod_container_status_restarts_total)" | head -c 300 && echo
```

If this returns data, the metric family exists.

---

## 5. Apply Manifests Correctly (Kustomize)

### 5.1 Common failure mode
If a directory contains `kustomization.yaml` and you apply it with `-f`, Kubernetes attempts to treat `kustomization.yaml` as a Kubernetes resource, which fails.

Incorrect:
```bash
kubectl apply -f analysis/dev
```

Correct:
```bash
kubectl apply -k analysis/dev
kubectl apply -k kustomize/overlays/dev
```

---

## 6. Verify Installation Result

### 6.1 Verify AnalysisTemplate exists
```bash
kubectl get analysistemplate -n project8-dev
```

### 6.2 Verify Rollout exists
```bash
kubectl get rollout -n project8-dev
```

### 6.3 Verify analysis step is present in the applied Rollout
```bash
kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml | grep -n "analysis:"
```

If analysis is missing, your dev overlay patch did not apply (see `docs/troubleshooting.md`).

---

## 7. Trigger a New Revision

AnalysisRuns are created when the rollout attempts a **new revision**.

If there is no change to the Rollout pod template (for example, the image tag didn’t change), you may not see a new AnalysisRun.

Details are in `docs/rollout-execution.md`.
