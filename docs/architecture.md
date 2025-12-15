# Architecture

## 1. Overview

Project 8 implements **progressive delivery** using Argo Rollouts. Instead of deploying a new version all at once, the platform:

1. Releases a new revision (new ReplicaSet) gradually
2. Pauses to observe behavior
3. Runs an automated **metric-based analysis**
4. Promotes on success or rolls back on failure

This is a production pattern used to reduce blast radius and convert deployments into controlled experiments.

---

## 2. Components

### 2.1 Argo CD (GitOps)
Argo CD continuously reconciles this repository into the cluster. Git is the source of truth for:

- Rollout objects (base + overlays)
- AnalysisTemplate resources
- Environment namespaces and overlays
- Argo CD Application definitions (if used)

### 2.2 Argo Rollouts (Progressive Delivery)
Argo Rollouts introduces:

- `Rollout` resources to replace standard `Deployment` rollouts
- Canary strategies with weight steps and pauses
- `AnalysisRun` execution integrated into rollout steps
- Automatic abort and rollback on failed analysis

### 2.3 Prometheus (Metrics Store + Query API)
Argo Rollouts can call Prometheus via HTTP to evaluate a rollout gate. Prometheus provides:

- The metric database (time series)
- The query language (PromQL)
- An HTTP endpoint for queries executed by AnalysisRuns

### 2.4 Kustomize (Base + Overlays)
Kustomize keeps shared manifests in a base and applies environment-specific deltas via overlays:

- `kustomize/base` defines the core Rollout + Services
- `kustomize/overlays/dev` injects the analysis step via a patch
- `analysis/dev` defines the AnalysisTemplate for dev

---

## 3. End-to-End Flow

1. **CI publishes a new image** (Project 1).  
2. **GitOps updates manifests** (image tag or rollout changes).  
3. **Argo CD syncs** the new desired state.  
4. **Argo Rollouts creates a new revision** (new ReplicaSet).  
5. Rollout shifts traffic to canary and pauses.  
6. Rollout executes an **AnalysisRun**:
   - Prometheus query is executed repeatedly on an interval
   - Results are evaluated against success/failure conditions
7. If analysis passes, rollout proceeds. If analysis fails, rollout aborts and rolls back.

---

## 4. Diagram

See: `docs/images/project8-architecture.svg`

---

## 5. Why This Matters (Interview-Level Framing)

Kubernetes alone can “roll forward” based on readiness probes, but readiness is not the same as user impact. Progressive delivery adds a missing layer:

- Deploy slowly
- Measure behavior
- Decide automatically
- Roll back safely

This project shows the engineering judgment required when “ideal metrics” are not available (no HTTP success rate),
and how to select a reliable gate from the telemetry that *does* exist.
