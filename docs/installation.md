# Installation and Prerequisites

This document captures everything required for **Argo Rollouts AnalysisTemplates** to work end-to-end in this project.

The main lesson: progressive delivery fails in confusing ways if you skip prerequisite validation. We explicitly validate:

- Argo Rollouts controller + CRDs
- Prometheus accessibility
- Metric existence (no “guessing”)
- Correct Kustomize apply method

---

## 1. Prerequisites

### 1.1 Kubernetes cluster access
You need working cluster connectivity:

```bash
kubectl cluster-info
kubectl get nodes
```

### 1.2 Namespaces
This project assumes environment namespaces exist (example: `project8-dev`, `project8-pre`, `project8-prod`).

Verify dev namespace exists:

```bash
kubectl get ns project8-dev
```

### 1.3 Argo CD (GitOps) installed
Argo CD is assumed to be present and configured to sync this repo. If not using Argo CD, you can apply manifests manually via `kubectl apply -k`.

---

## 2. Install Argo Rollouts

### 2.1 What must be installed
Argo Rollouts includes:

- The rollouts controller (Deployment in `argo-rollouts` namespace, by default)
- CRDs for:
  - `Rollout`
  - `AnalysisTemplate`
  - `AnalysisRun`
  - (plus other resources, depending on version)

### 2.2 Verify CRDs (critical)
If these CRDs do not exist, you cannot create the objects used in this project:

```bash
kubectl get crd | grep argoproj.io | grep -E "rollouts|analysistemplates|analysisruns"
```

Expected: CRDs matching `rollouts.argoproj.io`, `analysistemplates.argoproj.io`, `analysisruns.argoproj.io`.

### 2.3 Verify the controller is running
```bash
kubectl get pods -n argo-rollouts
```

You should see a running rollouts controller pod.

---

## 3. Prometheus Requirements

### 3.1 Prometheus must exist
This project assumes Prometheus is deployed (commonly via kube-prometheus-stack from Project 7).

Verify Prometheus Service exists (example naming):

```bash
kubectl get svc -n monitoring | grep prometheus
```

### 3.2 Prometheus must be reachable by the Rollouts controller
AnalysisRuns execute queries server-side (inside the cluster). Prometheus must be reachable via a cluster DNS name such as:

- `http://kube-prometheus-stack-prometheus.monitoring:9090`

If Prometheus is unreachable, AnalysisRuns will error or fail.

---

## 4. Validate Prometheus Health and Metric Availability

This is a required pre-step before writing an AnalysisTemplate.

### 4.1 Port-forward (local validation)
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

### 4.2 Baseline query must return data
The `up` metric should return results if Prometheus is healthy:

```bash
curl -s "http://localhost:9090/api/v1/query?query=up" | head -c 400 && echo
```

### 4.3 Validate the gate metric exists
Project 8 uses kube-state-metrics restart counters:

```bash
curl -s "http://localhost:9090/api/v1/query?query=count(kube_pod_container_status_restarts_total)" | head -c 300 && echo
```

If this returns results, the metric exists.

---

## 5. Apply Manifests Correctly (Kustomize)

### 5.1 The common failure mode
If you run:

```bash
kubectl apply -f analysis/dev
```

and that directory contains `kustomization.yaml`, Kubernetes will try to apply it as if it were a Kubernetes resource.
This fails with an error like “no matches for kind Kustomization…” because **Kustomize’s Kustomization is not a Kubernetes CRD**.

### 5.2 Correct usage
Use `-k` to apply a Kustomize directory:

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

### 6.3 Verify Rollout includes an analysis step
```bash
kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml | grep -n "analysis:"
```

If analysis is missing, it usually means the dev overlay patch did not apply correctly (see `docs/troubleshooting.md`).

---

## 7. Final Prerequisite: Trigger a New Revision

AnalysisRuns are created during rollout progression for a **new revision**.

If there is no new ReplicaSet (no new pod template hash), you may not see new AnalysisRuns.
Triggering a new revision typically means updating the image tag (usually done by CI) or changing the Rollout pod template.

Verification is covered in `docs/rollout-execution.md`.
