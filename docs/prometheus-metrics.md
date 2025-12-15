# Prometheus Metrics and PromQL Derivation

This document explains:

1. How we validated which metrics were available
2. Why HTTP success-rate metrics were not usable in this environment
3. How we selected a reliable “default available” metric
4. How the final PromQL query was derived (step-by-step)

---

## 1. Start With Reality: “Does the Metric Exist?”

Prometheus does not return metrics that are not scraped or produced.

When you query a metric family that does not exist, Prometheus returns an empty vector (`[]`).
This is not an error — it’s a valid response indicating “no series matched.”

In this project, HTTP-style metrics (e.g., `http_requests_total`) were not present in Prometheus, which made request-based success rate gates impossible without additional instrumentation and scraping configuration.

---

## 2. Baseline Prometheus Validation

Before trusting any analysis gate, validate Prometheus itself:

### 2.1 `up` (baseline health metric)
```bash
curl -s "http://localhost:9090/api/v1/query?query=up" | head -c 400 && echo
```

If `up` returns results, Prometheus is healthy and scraping targets.

---

## 3. Candidate Metric Validation

### 3.1 “Guessing” metrics is expensive
A rollout gate must use **proven** metrics. The workflow is:

1. Query metric family existence via `count(<metric>)`
2. If it returns `[]`, stop and choose another metric or install instrumentation

Example: “does http_requests_total exist?”

```bash
curl -s "http://localhost:9090/api/v1/query?query=count(http_requests_total)"
```

If result is `[]`, you do not have that metric in this environment.

---

## 4. Why We Chose Restart-Based Metrics

When app-level metrics were absent, we chose a metric that is typically present when kube-state-metrics is installed:

- `kube_pod_container_status_restarts_total`

This is valuable because:
- it detects instability that users would experience (crashes, restarts)
- it does not require application instrumentation
- it is safe to use as a conservative gate

---

## 5. The Final Query

```promql
sum(increase(kube_pod_container_status_restarts_total{namespace="project8-dev"}[2m]))
```

This returns a single number: **how many container restarts occurred in the last 2 minutes in the dev namespace**.

---

## 6. Query Breakdown (Explain It Like You Own It)

### 6.1 `kube_pod_container_status_restarts_total`
This is a **counter** that increases each time a container restarts.

Key counter property:
- it only increases (except on reset due to scrape target restart)
- raw counter values are not directly meaningful for “what happened recently”
- you compute changes over time windows

### 6.2 `{namespace="project8-dev"}`
This scoping is critical. Without it:
- restarts in unrelated namespaces could fail your rollout
- your gate becomes noisy and non-actionable

By scoping to the rollout namespace, you measure only what the rollout can plausibly influence.

### 6.3 `[2m]` range vector
This asks Prometheus: “give me the series values over the last 2 minutes.”

You need a range vector because counters represent cumulative history — you want a *recent* window.

### 6.4 `increase(counter[2m])`
`increase()` calculates how much the counter increased over the time range.

For restarts, this approximates “how many restarts occurred in the window.”
It is the correct function for converting a counter into a windowed event count.

### 6.5 `sum(...)`
In a namespace there can be many pods and containers. `sum()` collapses all the individual increases into a single scalar.

That scalar is ideal for gating conditions because:
- it is easy to interpret (0 vs >0)
- it avoids label cardinality complexity in the rollout gate

---

## 7. How This Becomes a Rollout Gate

The AnalysisTemplate interprets the query result as:
- success if the value exists and equals 0
- failure if it’s missing or > 0

This is a deliberate “safety-first” policy: *we promote only when we can prove stability.*

---

## 8. Future Enhancement (Optional)

If you wanted a request-based success rate gate in a future iteration, you would need:
- application metrics (instrumented endpoint) and a ServiceMonitor, or
- ingress/proxy metrics known to exist and be scraped

Project 8 intentionally documents the realistic case: you don’t always have perfect metrics on day 1, so you design with what the platform provides.
