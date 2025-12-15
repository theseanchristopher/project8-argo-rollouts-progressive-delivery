# Troubleshooting

This document captures the primary failure modes encountered in Project 8 and the reliable debugging workflow.
Format: **Symptom → Likely Cause → Fix / Verification**.

---

## 1. “No AnalysisRun created”

### Symptom
You trigger a rollout, but `kubectl get analysisruns -n project8-dev` shows nothing new.

### Likely causes
- No new rollout revision was created (pod template unchanged)
- Rollout never reached the analysis step (paused earlier or aborted)
- Dev overlay patch that injects analysis did not apply
- AnalysisTemplate does not exist in the namespace

### Fix / verify
1. Confirm revision increment and ReplicaSet creation:
   ```bash
   kubectl get rs -n project8-dev --sort-by=.metadata.creationTimestamp
   ```
2. Confirm analysis step exists in applied Rollout:
   ```bash
   kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml | grep -n "analysis:"
   ```
3. Confirm AnalysisTemplate exists:
   ```bash
   kubectl get analysistemplate -n project8-dev
   ```
4. Trigger a new revision (change image tag or pod template).

---

## 2. “AnalysisRun Error: reflect: slice index out of range”

### Symptom
AnalysisRun fails with an error message indicating an index error.

### Likely cause
Your `successCondition` references `result[0]` but Prometheus returned an empty vector (`[]`).

### Fix / verify
- Make success/failure conditions defensive:
  - fail if `len(result) == 0`
  - only evaluate `result[0]` when `len(result) > 0`
- Verify by describing the AnalysisRun and checking “Value: []” in measurements.

---

## 3. “AnalysisRun returns Value: [] (empty vector)”

### Symptom
AnalysisRun runs, but measurements show `Value: []` and the run fails.

### Likely causes
- Metric family doesn’t exist in Prometheus
- Label selectors match nothing (wrong namespace/labels)
- Prometheus is reachable, but the target metric is not scraped

### Fix / verify
1. Validate Prometheus health:
   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=up" | head -c 200 && echo
   ```
2. Validate metric existence:
   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=count(kube_pod_container_status_restarts_total)"
   ```
3. Validate your final query returns a scalar:
   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=sum(increase(kube_pod_container_status_restarts_total%7Bnamespace%3D%22project8-dev%22%7D%5B2m%5D))"
   ```

---

## 4. “kubectl apply -f analysis/dev fails with Kustomization kind error”

### Symptom
You applied a folder containing `kustomization.yaml` using `-f` and got an error such as:
- “no matches for kind Kustomization in version kustomize.config.k8s.io/v1beta1”

### Likely cause
Kustomize `kustomization.yaml` is not a Kubernetes object; `kubectl apply -f` tries to apply it as YAML.

### Fix
Use `-k`:
```bash
kubectl apply -k analysis/dev
kubectl apply -k kustomize/overlays/dev
```

---

## 5. “Kustomize patch failed: failed to find unique target for patch”

### Symptom
Applying `kubectl apply -k kustomize/overlays/dev` fails with a patch-target error.

### Likely causes
- Patch references wrong resource name
- Namespace mismatch between patch and base
- Base resource kind/apiVersion differs from what the patch targets

### Fix / verify
- Render the overlay output and confirm the Rollout resource name/namespace match your patch:
  ```bash
  kubectl kustomize kustomize/overlays/dev | head -n 80
  ```
- Confirm patch `metadata.name` matches the base Rollout name exactly.

---

## 6. “Rollout didn’t create a new revision”

### Symptom
You expected a canary run, but rollout stays on the same revision.

### Likely cause
No pod template change occurred.

### Fix / verify
- Update the image tag (most common), or
- change a pod template annotation, then sync
- confirm a new ReplicaSet exists:
  ```bash
  kubectl get rs -n project8-dev --sort-by=.metadata.creationTimestamp
  ```

---

## 7. “Prometheus query works locally but fails in AnalysisRun”

### Symptom
Your port-forwarded curl query works, but AnalysisRun returns empty or errors.

### Likely causes
- Different Prometheus address inside the cluster (wrong service DNS)
- Network policy or service name mismatch
- Prometheus is reachable from your laptop but not from rollouts controller

### Fix / verify
- Confirm the `provider.prometheus.address` points to the correct in-cluster DNS name
- Describe the AnalysisRun and check the “Resolved Prometheus Query” and provider address
- Confirm Prometheus service exists and resolves in cluster

---

## 8. Debug “golden path” (fastest reliable sequence)

1. Confirm Rollout + template exist:
   ```bash
   kubectl get rollout,analysistemplate -n project8-dev
   ```
2. Confirm analysis step exists in rollout YAML:
   ```bash
   kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml | grep -n "analysis:"
   ```
3. Trigger a new revision (change image tag).
4. Watch rollout and AnalysisRuns:
   ```bash
   kubectl argo rollouts get rollout project8-nginx-rollout -n project8-dev --watch
   kubectl get analysisruns -n project8-dev --sort-by=.metadata.creationTimestamp
   ```
5. If failed, describe the AnalysisRun and read the resolved query + measurement values.
