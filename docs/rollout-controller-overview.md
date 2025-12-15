# Rollout Controller Overview (Argo Rollouts)

This document explains the Argo Rollouts mental model: how Rollouts work, what a “revision” means,
and when AnalysisRuns are created.

---

## 1. Why Rollouts Instead of Deployments

A standard Kubernetes `Deployment` supports rollout strategies (RollingUpdate), but it cannot natively:

- Shift traffic by percentage using stable/canary services
- Pause between traffic changes for observation
- Run metric-driven experiments during rollout steps
- Automatically abort and roll back based on custom metrics

Argo Rollouts fills this gap.

---

## 2. Core Objects

### 2.1 Rollout
A `Rollout` is a controller-managed resource that behaves like a `Deployment`, but with advanced strategy support.

In this project, the Rollout:
- creates ReplicaSets (revisions)
- shifts traffic using stable/canary service selectors
- executes analysis steps

### 2.2 ReplicaSet Revisions
A new revision is created when the Rollout’s **pod template** changes (commonly: image tag change). Each revision gets a new “pod-template-hash.”

Practical implication:
- If you don’t change the pod template, you won’t trigger a new revision
- If you don’t trigger a new revision, you may not see a new AnalysisRun

### 2.3 Services (Stable / Canary)
This project uses two services:
- **stable service** selects stable ReplicaSet
- **canary service** selects canary ReplicaSet

Traffic management depends on the chosen traffic routing method. Even without an ingress/router integration, the Rollout still executes steps and analysis, but “traffic” in practice may be conceptual unless routed through a layer that honors the canary/stable split.

### 2.4 AnalysisTemplate and AnalysisRun
- `AnalysisTemplate` defines *what to measure* and *how to interpret it*
- `AnalysisRun` is an instantiation created during a rollout step

---

## 3. Canary Steps and What They Mean

A canary strategy is defined as a sequence of steps, typically:

- setWeight: shift a percentage to canary
- pause: wait for a period
- analysis: run a defined analysis gate

Rollouts executes steps in order. If any analysis fails:
- the rollout aborts
- the system rolls back to stable revision

---

## 4. When AnalysisRuns Are Created

AnalysisRuns are created when the rollout reaches an `analysis` step for an active rollout attempt.

Common reasons you may not see a new AnalysisRun:
- no new revision was created
- rollout never reached the analysis step (paused earlier, or aborted early)
- patch that injects the analysis step did not apply
- namespace mismatch (template not found)

---

## 5. Operational Commands

### 5.1 Watch rollout status
```bash
kubectl argo rollouts get rollout project8-nginx-rollout -n project8-dev --watch
```

### 5.2 List revisions and ReplicaSets
```bash
kubectl get rs -n project8-dev --sort-by=.metadata.creationTimestamp
```

### 5.3 Inspect rollout YAML for step configuration
```bash
kubectl get rollout project8-nginx-rollout -n project8-dev -o yaml
```

---

## 6. Interview Summary

Argo Rollouts adds a control plane for deployments:
- you roll out incrementally
- you pause and measure
- you decide automatically
- you roll back safely

That pattern (progressive delivery) is one of the clearest signals that a candidate understands real production deployment engineering.
