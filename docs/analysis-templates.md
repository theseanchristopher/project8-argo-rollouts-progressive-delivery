# AnalysisTemplates and AnalysisRuns

This document explains **how Argo Rollouts uses AnalysisTemplates**, how metrics are evaluated, and how to design conditions that fail safely.

---

## 1. What an AnalysisTemplate Is

An `AnalysisTemplate` defines:

- which metrics to query (and how often)
- what counts as success or failure
- how many failures are tolerated before the analysis run fails
- which provider to use (Prometheus in this project)

Think of it as: **a repeatable, declarative experiment definition**.

---

## 2. What an AnalysisRun Is

An `AnalysisRun` is created when:
- a Rollout hits an analysis step, or
- an analysis is triggered in another supported way

The AnalysisRun contains:
- the resolved Prometheus query
- individual measurements (each query execution)
- final phase: Successful / Failed / Error

---

## 3. Key Fields You Must Understand

### 3.1 `interval`
How often the metric query runs.

Example: `interval: 30s` means one measurement every 30 seconds.

### 3.2 `count`
How many measurements to take.

Example: `count: 5` with `interval: 30s` means the analysis window is ~2.5 minutes.

### 3.3 `failureLimit`
How many failed measurements are allowed before the AnalysisRun fails.

Example: `failureLimit: 1` makes the gate sensitive to instability.

### 3.4 `successCondition` and `failureCondition`
These are boolean expressions evaluated against the Prometheus query result vector, exposed as `result`.

Important: Prometheus can return an empty vector (`[]`). If you write a condition like:

```text
result[0] >= 0.95
```

and `result` is empty, the AnalysisRun can error (index out of range).

In this project, we defensively handle empty results.

---

## 4. Fail-Closed vs Fail-Open

### Fail-open (not recommended for safety gates)
If missing data counts as success, you risk promoting unhealthy builds when the monitoring system is misconfigured or unreachable.

### Fail-closed (recommended for safety gates)
If missing data counts as failure, you prevent promotion unless the system can prove health.

Project 8 uses **fail-closed** behavior.

---

## 5. Defensive Conditions Used in This Project

We use defensive conditions that safely handle empty vectors:

- Failure if `len(result) == 0` (missing data)
- Failure if value indicates instability (restarts > 0)
- Success only if a value exists AND indicates health

This eliminates “slice index out of range” failures and converts “no data” into a clear failure reason.

---

## 6. How to Inspect an AnalysisRun

List recent AnalysisRuns:

```bash
kubectl get analysisruns -n project8-dev --sort-by=.metadata.creationTimestamp
```

Describe one (most useful command):

```bash
kubectl describe analysisrun -n project8-dev <NAME>
```

Look for:
- resolved Prometheus query
- measurement values
- final message (why it failed)

---

## 7. Practical Interview Framing

An AnalysisTemplate is “deployment policy as code.” It encodes:

- what “healthy” means
- what signals to trust
- how conservative to be under uncertainty
- how to prevent unobserved promotion

This is exactly how modern SRE/Platform teams prevent bad releases from reaching users.
