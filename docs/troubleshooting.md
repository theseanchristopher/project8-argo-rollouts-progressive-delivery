# Troubleshooting

This document captures the primary failure modes encountered in Project 8 and the reliable debugging workflow.

Format: **Symptom → Likely Cause → Fix / Verification**

---

## 1. No AnalysisRun Created

### Symptom
You trigger a rollout, but `kubectl get analysisruns -n project8-dev` shows nothing new.

### Likely causes
- No new rollout revision was created (pod template unchanged)
- Rollout never reached the analysis step (paused earlier or aborted)
- Dev overlay patch that injects analysis did not apply
- AnalysisTemplate does not exist in the namespace

### Fix / verify
1. Confirm a new ReplicaSet exists (revision increment):
   ```bash
   kubectl get rs -n project8-dev --sort-by=.metadata.creationTimestamp
   ```
2. Confirm analysis step exists in the applied Rollout:
   ```bash
   kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml | grep -n "analysis:"
   ```
3. Confirm AnalysisTemplate exists:
   ```bash
   kubectl get analysistemplate -n project8-dev
   ```
4. Trigger a new revision (update image tag or pod template).

---

## 2. AnalysisRun Error: “slice index out of range”

### Symptom
AnalysisRun fails with an error indicating an index/slice error.

### Likely cause
Your `successCondition` references `result[0]`, but Prometheus returned an empty vector (`[]`).

### Fix / verify
- Make conditions defensive:
  - fail if `len(result) == 0`
  - only evaluate `result[0]` when `len(result) > 0`
- Verify by describing the AnalysisRun and checking measurement values.

---

## 3. AnalysisRun Returns `Value: []` (Empty Vector)

### Symptom
AnalysisRun runs, but measurements show `Value: []` and the run fails.

### Likely causes
- Metric family doesn’t exist in Prometheus
- Label selectors match nothing (wrong namespace/labels)
- Prometheus is reachable but kube-state-metrics isn’t installed/scraped

### Fix / verify
1. Validate Prometheus health:
   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=up" | head -c 200 && echo
   ```
2. Validate metric existence:
   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=count(kube_pod_container_status_restarts_total)"
   ```
3. Validate the final query returns a scalar:
   ```bash
   curl -s "http://localhost:9090/api/v1/query?query=sum(increase(kube_pod_container_status_restarts_total%7Bnamespace%3D%22project8-dev%22%7D%5B2m%5D))"
   ```

---

## 4. `kubectl apply -f` Against a Kustomize Directory Fails

### Symptom
You applied a folder containing `kustomization.yaml` with `kubectl apply -f` and got an error like:
- “no matches for kind Kustomization…”

### Likely cause
Kustomize `kustomization.yaml` is not a Kubernetes resource; `kubectl apply -f` tries to apply it as YAML.

### Fix
Use `-k`:

```bash
kubectl apply -k analysis/dev
kubectl apply -k kustomize/overlays/dev
```

---

## 5. Kustomize Patch Error: “failed to find unique target for patch”

### Symptom
Applying `kubectl apply -k kustomize/overlays/dev` fails with a patch-target error.

### Likely causes
- Patch references the wrong resource name
- Namespace mismatch between patch and base
- Base resource kind/apiVersion differs from what the patch targets

### Fix / verify
- Render the overlay output and confirm the Rollout resource name/namespace match your patch:
  ```bash
  kubectl kustomize kustomize/overlays/dev | head -n 120
  ```
- Confirm patch `metadata.name` matches the base Rollout name exactly.

---

## 6. Prometheus Query Works Locally But Fails in AnalysisRun

### Symptom
Your port-forwarded query works, but AnalysisRun errors or returns empty.

### Likely causes
- AnalysisTemplate points to the wrong in-cluster Prometheus address
- Prometheus Service name/namespace mismatch
- Network policy blocks rollouts controller from reaching Prometheus

### Fix / verify
- Confirm `provider.prometheus.address` is the correct in-cluster DNS name.
- Describe the AnalysisRun and check the provider address and resolved query.
- Confirm the Prometheus Service exists in-cluster.

---

## 7. Debug Golden Path (Fastest Reliable Sequence)

1. Confirm Rollout + AnalysisTemplate exist:
   ```bash
   kubectl get rollout,analysistemplate -n project8-dev
   ```
2. Confirm analysis step exists in the applied Rollout:
   ```bash
   kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml | grep -n "analysis:"
   ```
3. Trigger a new revision (update image tag).
4. Watch rollout and AnalysisRuns:
   ```bash
   kubectl argo rollouts get rollout project8-nginx-rollout -n project8-dev --watch
   kubectl get analysisruns -n project8-dev --sort-by=.metadata.creationTimestamp
   ```
5. If failed, describe the AnalysisRun and read:
   - provider address
   - resolved query
   - measurement values
   - final message
