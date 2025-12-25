# Prometheus Metrics and PromQL Derivation

This document explains how Project 8 selected a rollout gate metric and derived the final PromQL query.

---

## 1. Start With Reality: Does the Metric Exist?

Prometheus only returns metrics that are being produced and scraped. If you query a metric family that does not exist, Prometheus returns an **empty vector** (`[]`).

That is not an error — it means “no series matched”.

For rollout gating, an empty vector is dangerous if you treat it as success.

---

## 2. Validate Prometheus Before Designing a Gate

Before trusting any analysis result, validate Prometheus itself.

Baseline query:

```bash
curl -s "http://localhost:9090/api/v1/query?query=up" | head -c 400 && echo
```

If `up` returns series, Prometheus is reachable and scraping.

---

## 3. Candidate Metric Discovery Workflow

“Guessing” metrics is expensive. The safe workflow is:

1. Pick a candidate metric family.
2. Check that it exists (returns series) using `count(<metric>)`.
3. If it returns `[]`, stop and choose another metric or add instrumentation.

Example (checking for request metrics):

```bash
curl -s "http://localhost:9090/api/v1/query?query=count(http_requests_total)"
```

If the response is `[]`, you do not have that metric in this environment.

---

## 4. Why Restart-Based Metrics Were Chosen

HTTP success-rate metrics were not available without adding instrumentation and scrape config.
To keep the project focused on rollout mechanics (not re-instrumenting the app), Project 8 uses a platform metric commonly present with kube-state-metrics:

- `kube_pod_container_status_restarts_total`

Why it’s a reasonable safety gate:

- Detects instability users experience (crashes / restarts)
- Requires no app changes
- Encourages conservative promotion behavior

---

## 5. The Final PromQL Query

```promql
sum(increase(kube_pod_container_status_restarts_total{namespace="project8-dev"}[2m]))
```

This returns a single scalar: **how many container restarts occurred in the last 2 minutes in the dev namespace**.

---

## 6. Query Breakdown

### 6.1 `kube_pod_container_status_restarts_total`
A **counter** that increments every time a container restarts.

Counter properties:
- monotonically increases (except on reset when the source restarts)
- the raw value is total history, not “recent events”

### 6.2 `{namespace="project8-dev"}`
Scopes the query to the rollout’s namespace so unrelated restarts don’t fail the gate.

Without this selector:
- restarts in other namespaces could incorrectly block promotion
- the signal becomes noisy and non-actionable

### 6.3 `[2m]` range vector
Counters require a time window to measure changes over time.

`[2m]` means “look at samples in the last 2 minutes”.

### 6.4 `increase(counter[2m])`
`increase()` calculates how much the counter increased over the window, which approximates the number of restart events in that period.

### 6.5 `sum(...)`
Collapses all series into a single number, ideal for success/failure conditions.

---

## 7. How This Becomes a Rollout Gate

The AnalysisTemplate interprets the result as:

- **Success:** a value exists AND equals `0`
- **Failure:** value is missing (`len(result)==0`) OR greater than `0`

This is an intentionally conservative policy: **promote only when stability is proven**.

---

## 8. Future Enhancement (Optional)

If you want a request-based success rate gate later, you would need:

- app instrumentation + `/metrics` + ServiceMonitor (Project 7 pattern), or
- ingress/proxy metrics known to exist and be scraped

Project 8 documents the realistic case: you often have to design gates using what the platform already provides.
