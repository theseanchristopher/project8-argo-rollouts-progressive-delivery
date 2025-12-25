# Project 8 — Progressive Delivery with Argo Rollouts and Prometheus (GitOps)

Project 8 demonstrates **production-style progressive delivery** on Kubernetes using:

- **Argo Rollouts** for canary deployments with step-based promotion
- **Prometheus-based analysis** to gate promotion on real telemetry
- **GitOps with Argo CD** so rollout policy and analysis gates are version-controlled

Instead of treating deployments as a binary “apply and hope” operation, this project turns a release into a **measured decision**:
shift gradually, pause, evaluate metrics, and **promote or roll back automatically**.

---

## 1. Problem Statement

A standard Kubernetes `Deployment` can roll forward using readiness probes and a RollingUpdate strategy, but it cannot answer:

- “Is the new revision stable after it becomes Ready?”
- “Did it start crash-looping 60 seconds after startup?”
- “Can we stop promotion automatically if telemetry is missing or unhealthy?”

Progressive delivery addresses these gaps by combining **incremental rollout steps** with **automated analysis**.

---

## 2. High-Level Architecture

This project ties together five layers:

1. **CI (Project 1)** builds and publishes container images
2. **GitOps repo (Project 8)** defines rollout strategy + analysis policy
3. **Argo CD** continuously syncs the desired state into the cluster
4. **Argo Rollouts** executes the canary strategy and analysis steps
5. **Prometheus** provides the metrics used for promotion decisions

Architecture diagram: `docs/images/project8-architecture.svg`

---

## 3. What This Project Demonstrates

- Replacing a Kubernetes `Deployment` with an Argo Rollouts **`Rollout`**
- A **step-based canary strategy** (weights + pauses)
- **AnalysisRuns** created and executed during a rollout step
- A **Prometheus provider** used directly by Argo Rollouts (no external scripts)
- A **fail-closed** analysis gate that treats missing telemetry as unsafe
- Automatic abort + rollback on failed analysis
- Debugging rollouts and analysis failures in a GitOps environment

---

## 4. Metrics Strategy (Real-World Constraints)

The “ideal” gate metric for many apps is an HTTP success rate (for example, a ratio of 2xx to total requests).
In this environment, request metrics were not available without adding instrumentation and additional scrape configuration.

Rather than inventing metrics, Project 8 uses a platform metric that is commonly present when `kube-state-metrics` is installed:

- `kube_pod_container_status_restarts_total`

The final gate measures **restart events in a recent window** and fails the rollout if restarts occur (or if the query returns no data).

PromQL used:

```promql
sum(increase(kube_pod_container_status_restarts_total{namespace="project8-dev"}[2m]))
```

---

## 5. Repository Structure

```text
project8-argo-rollouts-progressive-delivery/
├── README.md
├── applications/
│   ├── project8-rollouts-dev-app.yaml
│   ├── project8-rollouts-pre-app.yaml
│   └── project8-rollouts-prod-app.yaml
├── analysis/
│   └── dev/
│       ├── analysis-template-prometheus.yaml
│       └── kustomization.yaml
├── kustomize/
│   ├── base/
│   │   ├── rollout.yaml
│   │   ├── service-stable.yaml
│   │   ├── service-canary.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── namespace.yaml
│       │   ├── rollout-analysis-patch.yaml
│       │   └── kustomization.yaml
│       ├── pre/
│       │   ├── namespace.yaml
│       │   └── kustomization.yaml
│       └── prod/
│           ├── namespace.yaml
│           └── kustomization.yaml
└── docs/
    ├── architecture.md
    ├── installation.md
    ├── rollout-controller-overview.md
    ├── analysis-templates.md
    ├── prometheus-metrics.md
    ├── rollout-execution.md
    ├── failure-and-rollback.md
    ├── troubleshooting.md
    ├── references.md
    └── images/
        └── project8-architecture.svg
```

---

## 6. Documentation Map

- **`docs/architecture.md`** — components and end-to-end flow
- **`docs/installation.md`** — prerequisites and required CRDs/services
- **`docs/rollout-controller-overview.md`** — Rollout vs Deployment, revisions, services
- **`docs/analysis-templates.md`** — AnalysisTemplate/AnalysisRun mechanics + safe conditions
- **`docs/prometheus-metrics.md`** — metric discovery and PromQL derivation
- **`docs/rollout-execution.md`** — how to trigger and verify canary + analysis
- **`docs/failure-and-rollback.md`** — what happens on failure and how to confirm rollback
- **`docs/troubleshooting.md`** — common failure modes and a debugging “golden path”
- **`docs/references.md`** — official documentation links

---

## 7. Key Takeaways

- Progressive delivery converts deployments into measurable, automated decisions
- “No telemetry” is a failure signal for safety gates (fail-closed)
- Platform metrics can be valid rollout gates when app metrics are missing
- GitOps makes rollout policy reproducible and auditable
- Argo Rollouts integrates analysis directly into the rollout controller loop

---

## 8. References

See `docs/references.md`.
