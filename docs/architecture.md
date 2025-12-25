# Architecture

## 1. Overview

Project 8 implements **progressive delivery** using Argo Rollouts. Instead of updating all pods at once, the platform:

1. Creates a new **revision** (a new ReplicaSet) when the pod template changes (typically the image tag).
2. Shifts traffic incrementally to the canary revision using a step-based strategy.
3. Pauses between steps to allow observation.
4. Executes a **Prometheus-backed analysis** during the rollout.
5. Promotes on success or **aborts + rolls back** on failure.

This pattern reduces blast radius and makes rollout behavior **deterministic and repeatable**.

---

## 2. Components

### 2.1 Argo CD (GitOps)
Argo CD continuously reconciles this repository into the cluster. Git is the source of truth for:

- Rollout objects (base + overlays)
- Environment namespaces (`project8-dev`, `project8-pre`, `project8-prod`)
- AnalysisTemplates used by rollouts
- Argo CD Applications (optional, in `applications/`)

### 2.2 Argo Rollouts (Progressive Delivery Controller)
Argo Rollouts extends Kubernetes deployments by introducing:

- `Rollout` resources (a Deployment-like controller with advanced strategy)
- Canary steps (setWeight, pause, analysis)
- `AnalysisRun` execution integrated into the rollout controller loop
- Automatic abort and rollback based on analysis outcomes

### 2.3 Prometheus (Metrics Store + Query API)
Argo Rollouts queries Prometheus over HTTP as part of an AnalysisRun:

- Prometheus stores time series scraped from the cluster
- PromQL selects and transforms those series into values suitable for a gate
- AnalysisRuns evaluate those values against success/failure conditions

### 2.4 Kustomize (Base + Overlays)
Kustomize keeps shared manifests in a base and applies environment-specific differences via overlays:

- `kustomize/base` defines the core Rollout and stable/canary Services
- `kustomize/overlays/*` creates namespaces and environment-specific deltas
- `kustomize/overlays/dev` injects an **analysis step** via a strategic merge patch
- `analysis/dev` defines the AnalysisTemplate used by the dev rollout

---

## 3. End-to-End Flow

1. **CI publishes a new image** (Project 1).
2. **GitOps updates the desired state** (image tag or rollout spec).
3. **Argo CD syncs** the change into the cluster.
4. **Argo Rollouts detects a pod template change** and creates a new ReplicaSet (new revision).
5. The rollout executes canary steps:
   - shift weight to canary
   - pause
   - run analysis (creates an AnalysisRun)
6. **AnalysisRun queries Prometheus** repeatedly on an interval and evaluates results.
7. Outcome:
   - **Pass:** rollout proceeds to the next step and eventually promotes.
   - **Fail/Error:** rollout aborts and returns to the previous stable revision.

---

## 4. Diagram

See: `docs/images/project8-architecture.svg`

---

## 5. Why This Matters

Readiness probes answer “did the container start?” Progressive delivery answers “is the new revision safe under observation?”

This project demonstrates the engineering judgment required when:
- ideal metrics are not available
- you must choose a conservative, reliable gate
- missing telemetry must be handled safely (fail-closed)
