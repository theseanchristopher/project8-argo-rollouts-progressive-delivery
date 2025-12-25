# Rollout Controller Overview (Argo Rollouts)

This document explains the Argo Rollouts mental model used in Project 8: what a Rollout is, what a revision is, and how stable/canary services interact with a canary strategy.

---

## 1. Why Rollouts Instead of Deployments

A Kubernetes `Deployment` can perform a RollingUpdate, but it cannot natively:

- shift traffic by percentage using stable/canary services
- pause between traffic changes
- run metric-driven experiments during rollout steps
- automatically abort + roll back based on custom metrics

Argo Rollouts adds these capabilities through a dedicated controller.

---

## 2. Core Objects

### 2.1 `Rollout`
A `Rollout` is a controller-managed resource that behaves like a `Deployment`, but supports advanced strategies.

In Project 8, the Rollout:

- creates ReplicaSets (revisions)
- advances through canary steps (weight → pause → analysis)
- aborts and rolls back on failed analysis

### 2.2 ReplicaSets and Revisions
A new revision is created when the Rollout’s **pod template** changes (commonly the image tag). Each revision gets a new `pod-template-hash`.

Practical implications:

- No pod template change → no new revision
- No new revision → no new canary attempt → no new AnalysisRun

### 2.3 Stable and Canary Services
This project defines two Services:

- **stable service** selects the stable ReplicaSet
- **canary service** selects the canary ReplicaSet

Depending on the traffic routing method, these services can be used by a router/ingress to direct user traffic. Even without advanced traffic routing, Rollouts still performs step progression and analysis deterministically.

### 2.4 AnalysisTemplate and AnalysisRun
- `AnalysisTemplate` defines what to measure and how to decide
- `AnalysisRun` is created during rollout progression and records each measurement and the final outcome

---

## 3. Canary Steps Used in Project 8

A canary strategy is a sequence of steps, typically:

- `setWeight`: shift some percentage to canary
- `pause`: wait for observation (time-based pause or manual)
- `analysis`: execute a gate based on telemetry

Rollouts executes steps in order. If the analysis fails:

- the rollout aborts
- the system returns to the previous stable revision

---

## 4. When AnalysisRuns Are Created

AnalysisRuns are created when the rollout reaches an `analysis` step for an active rollout attempt.

Common reasons you may not see a new AnalysisRun:

- no new revision was created (pod template unchanged)
- the rollout never reached the analysis step (paused earlier or aborted)
- the analysis step patch did not apply in the dev overlay
- the AnalysisTemplate does not exist in the namespace

---

## 5. Operational Commands

### 5.1 Watch rollout status
```bash
kubectl argo rollouts get rollout project8-nginx-rollout -n project8-dev --watch
```

### 5.2 Inspect revisions (ReplicaSets)
```bash
kubectl get rs -n project8-dev --sort-by=.metadata.creationTimestamp
```

### 5.3 Inspect applied rollout YAML
```bash
kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml
```

---

## 6. Interview Summary

Argo Rollouts adds a control plane for deployments:

- roll out incrementally
- pause and measure
- decide automatically
- roll back safely

This project demonstrates not just tooling, but the decision-making pattern used in production platform engineering.
