# AnalysisTemplates and AnalysisRuns

This document explains how Argo Rollouts uses **AnalysisTemplates** and **AnalysisRuns**, and why Project 8 designs analysis conditions to fail safely.

---

## 1. What an AnalysisTemplate Is

An `AnalysisTemplate` is a reusable definition of:

- which metric(s) to query (and which provider to use)
- how often to query (`interval`) and how many times (`count`)
- what “success” and “failure” mean (conditions evaluated against results)
- how many failed measurements are tolerated (`failureLimit`)

Think of it as **deployment policy as code**.

---

## 2. What an AnalysisRun Is

An `AnalysisRun` is the runtime instantiation of an AnalysisTemplate. It is created when a Rollout reaches an `analysis` step.

The AnalysisRun contains:

- the resolved query and provider configuration
- a list of measurements (one per interval)
- the final phase: `Successful`, `Failed`, or `Error`
- a message describing *why* it ended in that phase

---

## 3. Key Fields Used in This Project

### 3.1 `interval`
How often the query runs.

Example:
- `interval: 30s` means one measurement every 30 seconds.

### 3.2 `count`
How many total measurements to take.

Example:
- `count: 5` with `interval: 30s` yields an analysis window of about 2.5 minutes.

### 3.3 `failureLimit`
How many failed measurements are allowed before the AnalysisRun fails.

Example:
- `failureLimit: 1` is conservative: instability fails the gate quickly.

### 3.4 `successCondition` and `failureCondition`
These are boolean expressions evaluated against the query result, exposed as `result`.

Important behavior:
- Prometheus can return an **empty vector** (`[]`).
- If your conditions assume `result[0]` always exists, an empty vector can produce errors (for example, “slice index out of range”).

---

## 4. Fail-Closed vs Fail-Open

### 4.1 Fail-open (not recommended for rollout safety)
Missing data counts as success.

Risk: the system promotes a bad release when monitoring is misconfigured or unreachable.

### 4.2 Fail-closed (recommended for rollout safety)
Missing data counts as failure.

Benefit: promotion only happens when the system can prove health.

Project 8 uses **fail-closed** behavior.

---

## 5. Defensive Conditions Used in Project 8

Project 8 uses conditions that:

- fail when `len(result) == 0` (no data / missing telemetry)
- fail when the value indicates instability (restarts > 0)
- succeed only when a value exists AND indicates stability

This converts ambiguous telemetry into a deterministic decision.

---

## 6. Inspecting an AnalysisRun

List AnalysisRuns:

```bash
kubectl get analysisruns -n project8-dev --sort-by=.metadata.creationTimestamp
```

Describe the latest run:

```bash
kubectl describe analysisrun -n project8-dev <NAME>
```

What to look for:
- **Resolved query** (what was executed)
- measurement values (including `Value: []` cases)
- the final message (fastest path to root cause)

---

## 7. Interview Framing

You can describe AnalysisTemplates as:

- “deployment policy as code”
- “automated guardrails that prevent unobserved promotion”
- “an SRE-style safety gate that is conservative under uncertainty”
