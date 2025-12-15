# Project 8 — Progressive Delivery with Argo Rollouts and Prometheus (GitOps)

This project demonstrates **production-grade progressive delivery** on Kubernetes using **Argo Rollouts**, **Prometheus-based analysis**, and **GitOps workflows with Argo CD**.

Rather than treating deployments as a binary “apply and hope” operation, Project 8 shows how deployments can be converted into **measured, automated decisions**. Each release is evaluated using real metrics before being promoted, and automatically rolled back if safety conditions are violated.

This project intentionally mirrors **real-world platform engineering constraints**, including imperfect metrics, missing signals, and the need to design **fail-safe deployment gates**.

---

## 1. Why Progressive Delivery Is Necessary

Standard Kubernetes `Deployment` rollouts rely on:
- Pod readiness probes
- RollingUpdate strategies

While useful, these mechanisms **do not measure user impact** or application stability beyond “the container started.” In production systems, many failures occur **after** a pod becomes ready:

- Crash loops shortly after startup
- Resource exhaustion under load
- Configuration errors that only surface minutes later

Progressive delivery addresses this gap by:
- Releasing changes incrementally
- Observing behavior between steps
- Making promotion decisions based on metrics
- Rolling back automatically when conditions degrade

Project 8 implements this model end to end.

---

## 2. High-Level Architecture

At a high level, Project 8 integrates five layers:

1. **CI (Project 1)** builds and publishes container images
2. **GitOps repository (Project 8)** defines rollout and analysis behavior
3. **Argo CD** continuously reconciles desired state into the cluster
4. **Argo Rollouts** manages canary deployments and analysis execution
5. **Prometheus** provides the metrics used to gate promotion

Each layer is loosely coupled and declarative, making failures observable and behavior reproducible.

A visual overview is provided in `docs/images/project8-architecture.svg`.

---

## 3. What This Project Demonstrates

This project intentionally focuses on **decision-making during deployments**, not just tooling.

Key capabilities demonstrated:

- Replacing a Kubernetes `Deployment` with an **Argo Rollouts `Rollout`**
- Implementing a **step-based canary strategy** with pauses
- Executing **AnalysisRuns** during a rollout step
- Querying Prometheus directly from Argo Rollouts
- Designing **defensive AnalysisTemplate conditions**
- Handling empty metric responses safely
- Automatically aborting and rolling back failed releases
- Debugging rollout and analysis failures in a GitOps environment

---

## 4. Metrics Strategy and Real-World Constraints

A central lesson of Project 8 is that **the metrics you want are not always the metrics you have**.

Initial attempts to gate the rollout using HTTP success-rate metrics failed because:
- The application was not instrumented
- Ingress/controller metrics were not available or not scraped
- Prometheus returned empty vectors for expected metric families

Rather than introducing new instrumentation mid-project, the rollout gate was redesigned using a **platform-level metric that was proven to exist**:

- `kube_pod_container_status_restarts_total` (from kube-state-metrics)

This metric detects instability (crashes, restarts) that directly correlates with user-facing failures, and is commonly available in Kubernetes clusters.

Project 8 documents:
- How metrics were validated before use
- Why empty Prometheus responses must be treated as failures
- How counter semantics and `increase()` were applied correctly
- Why a fail-closed gate is safer than fail-open behavior

---

## 5. Canary Rollout and Analysis Flow

The rollout follows a step-based canary pattern:

1. A new revision is created (new ReplicaSet)
2. Traffic is shifted partially to the canary
3. The rollout pauses for observation
4. An **AnalysisRun** executes:
   - Prometheus is queried on an interval
   - Results are evaluated against success/failure conditions
5. On success, the rollout continues
6. On failure, the rollout aborts and rolls back

Analysis is not a sidecar or external script — it is **part of the rollout controller’s decision loop**.

---

## 6. Failure Handling and Automatic Rollback

Failure is treated as a **first-class outcome**, not an exception.

When analysis fails:
- The rollout stops progressing
- Traffic returns to the stable revision
- Canary pods are scaled down
- The failure is recorded for audit and debugging

This behavior is deterministic and repeatable, which is critical for production systems where “partial success” is not acceptable.

---

## 7. Repository Structure

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
│       └── prod/
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

## 8. Documentation Map

The `/docs` directory contains deep, standalone documentation:

- **architecture.md** — system overview and component interactions
- **installation.md** — prerequisites and required components
- **rollout-controller-overview.md** — how Argo Rollouts works internally
- **analysis-templates.md** — AnalysisTemplate and AnalysisRun behavior
- **prometheus-metrics.md** — metric discovery and PromQL derivation
- **rollout-execution.md** — how to run and verify canary deployments
- **failure-and-rollback.md** — rollback mechanics and safety guarantees
- **troubleshooting.md** — common failure modes and debugging workflows
- **references.md** — official documentation links

---

## 9. Key Takeaways

- Progressive delivery converts deployments into measurable decisions
- Metrics must be validated before they can be trusted
- Empty telemetry is a failure signal, not a success signal
- Rollouts should fail safely by default
- GitOps + Argo Rollouts provides a repeatable, auditable deployment model

---

## 10. References

See `docs/references.md` for official documentation used throughout this project.
