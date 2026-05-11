# Day 23 Lab Reflection

**Student:** Son
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/sines05/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.0)
Compose v2:    OK  (5.1.3)
RAM available: 14.81 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /home/son/Day23-Track2-Observability-Lab/00-setup/setup-report.json
```

The setup report confirms Docker 29.4.0, Compose v2 5.1.3, 14.81 GB RAM available (above the 8 GB minimum), and all 9 required ports were free before stack startup. The `setup-report.json` was committed to `00-setup/`.

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

The **AI Service Overview (Day 23)** dashboard contains 6 panels:
1. **Request Rate (RPS) by status** — timeseries showing `ok` and `error` rates
2. **Latency P50 / P95 / P99** — timeseries from histogram quantiles
3. **Error Rate (last 5m)** — stat panel with red threshold at 5%
4. **GPU Utilization** — gauge panel [0,100]
5. **Token Throughput (in/out per sec)** — timeseries from counter rate
6. **In-Flight Requests** — stat panel from `inference_active_gauge`

After generating 1,202 requests, all 6 panels populated with real data:
- Request rate: ~20 RPS during load
- P99 latency: ~0.25 s (all within the 0.5 s SLO bucket)
- Error rate: 0% (all `ok` status)
- Token throughput: 8,527 input tokens / 36,064 output tokens processed

Screenshot: `submission/screenshots/dashboard-overview.png`

### Burn-rate panel

The **SLO Burn Rate (Day 23)** dashboard tracks error budget consumption at four windows (5m, 30m, 1h, 6h). With 0% error rate during the load test, all burn rates registered at 0 — confirming the SLO (99.5% success) was not breached. The "Error Budget Remaining" stat showed 100%.

Screenshot: `submission/screenshots/slo-burn-rate.png`

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | `make alert` triggered: `docker stop day23-app` | `scripts/trigger-alert.sh` ran |
| T0+60s | `ServiceDown` fired in Alertmanager | screenshot `alertmanager-firing.png` |
| T0+90s | Slack received `ServiceDown FIRING` notification | screenshot `slack-firing.png` |
| T1+30s | App restored: `docker start day23-app` | — |
| T1+90s | Alert resolved | screenshot `slack-resolved.png` |

The alertmanager.yml routes `ServiceDown` severity=critical to the Slack webhook. The inhibit rule suppresses child alerts when `ServiceDown` fires, which prevented cascading alert noise.

### One thing surprised me about Prometheus / Grafana

I was surprised that Grafana's dashboard provisioning `datasources.yml` does **not** auto-set explicit UIDs by default — Grafana generates random UUIDs (e.g., `PBFA97CFB590B2093`) at first startup. This means any dashboard JSON that hardcodes `"uid": "prometheus"` will silently show "No Data" until the datasource UID is forced in provisioning. Adding `uid: prometheus` to `datasources.yml` and restarting Grafana resolved this — a subtle but critical configuration detail that isn't obvious from the docs.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

The Jaeger UI at `http://localhost:16686` shows traces from `inference-api`. A `POST /predict` request produces spans:
- `POST /predict` (root — FastAPI auto-instrumentation)
  - `embed-text` — `text.length` attribute, ~5 ms
  - `vector-search` — `k=5` attribute, ~10 ms
  - `generate-tokens` — `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reason` attributes

Screenshot: `submission/screenshots/jaeger-trace.png`

### Log line correlated to trace

Sample structured JSON log from the app (retrieved from Loki):

```json
{"event": "prediction served", "level": "info", "timestamp": "2026-05-11T16:33:01.412Z", "model": "llama3-mock", "input_tokens": 4, "output_tokens": 11, "quality": 0.834, "duration_seconds": 0.1783, "trace_id": "d941af887dec886aec32c50c5b8ea1c7"}
```

The `trace_id` field (`d941af887dec886aec32c50c5b8ea1c7`) correlates directly to the Jaeger trace for that request. The Loki datasource in Grafana has a derived field regex `"trace_id":"([a-fA-F0-9]+)"` that auto-links log lines to Jaeger spans — enabling seamless log-to-trace navigation.

### Tail-sampling math

The OTel Collector's `tail_sampling` processor has three policies evaluated in order:
1. **keep-errors** — 100% of ERROR-status traces retained
2. **keep-slow** — 100% of traces with latency > 2,000 ms retained
3. **probabilistic-1pct** — 1% of remaining healthy/fast traces retained

With the mock inference service generating ~20 RPS and P99 well under 2 s:
- Errors: ~0 traces/sec (fail=false in load test)
- Slow (>2 s): ~0 traces/sec
- Healthy eligible: 20 traces/sec × 1% = **0.2 traces/sec retained** from healthy traffic

So the policy keeps essentially **only error + slow traces** (critical signal), plus a statistical 1% sample for baselining. The `decision_wait: 30 s` buffer means the collector holds 50,000 trace buffers in memory while waiting to see if a slow or error span arrives before deciding.

A forced-error request (`fail=true`) triggers the `keep-errors` policy and is **guaranteed** to appear in Jaeger within 30 s. A typical fast healthy request has only a 1% chance of appearing. This is the fundamental value proposition of tail sampling: the storage/bandwidth cost scales with *signal*, not with *throughput*.

---

## 4. Track 04 — Drift Detection

### PSI scores

From `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

Two features drifted: `prompt_length` (PSI=3.461, extreme) and `response_quality` (PSI=8.85, extreme). `embedding_norm` and `response_length` remained stable (PSI < 0.1 threshold).

### Which test fits which feature?

| Feature | Recommended Production Test | Rationale |
|---|---|---|
| `prompt_length` | **KS (Kolmogorov-Smirnov)** | Continuous unbounded integer distribution. KS tests the CDF shift without assuming normality. PSI is useful but requires binning choices; KS is assumption-free for continuous data. |
| `embedding_norm` | **MMD (Maximum Mean Discrepancy)** | High-dimensional floating-point feature from a neural encoder. KS and PSI lose power in high dimensions. MMD with an RBF kernel captures distributional differences in embedding space natively — ideal for semantic drift detection. |
| `response_length` | **PSI (Population Stability Index)** | Discrete-ish token count with a meaningful reference distribution from production baseline. PSI gives an intuitive "how different is today's traffic?" score that business stakeholders can understand and threshold. |
| `response_quality` | **KL Divergence** | A bounded [0,1] quality score from an evaluator model — essentially a probability-like quantity. KL divergence measures information-theoretic distance between the baseline quality distribution and current, making it sensitive to shifts in both tails (quality degradation OR quality spikes). |

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The **Day 17 — Airflow DAG Duration** metric would be hardest to expose in a cross-day integration. Airflow uses its own internal metrics system (StatsD) that requires configuring a StatsD exporter sidecar and aligning job names across environments. Unlike Day 19 (Qdrant exposes a `/metrics` endpoint natively) or Day 20 (llama.cpp has a built-in Prometheus scrape target), Airflow's DAG execution metrics only appear after DAG runs complete — they're event-driven, not time-series polled. This means you can't simply point Prometheus at a static endpoint; you need a StatsD-to-Prometheus bridge and careful label alignment (`dag_id`, `run_id`) to correlate with downstream service health.

For this lab, the Cross-Day dashboard (`full-stack-dashboard.json`) was built with 6 panels that gracefully display "No Data" when prior-day services aren't running. All 6 panels render correctly with the `noValue` fallback text.

---

## 6. The single change that mattered most

**Fixing the Grafana datasource UID to enable dashboard-to-data connectivity.**

The most impactful single change in building this observability stack was adding `uid: prometheus` explicitly to the Grafana datasource provisioning YAML. Every dashboard JSON panel uses `"datasource": {"type": "prometheus", "uid": "prometheus"}` — but by default, Grafana generates a random UUID for auto-provisioned datasources (e.g., `PBFA97CFB590B2093`). This is a silent failure: dashboards load without error messages, panels render with loading spinners, and then show "No Data" — exactly mimicking the behavior you'd expect from a scrape misconfiguration or a data gap.

This connects directly to the **"cardinality and data model" concepts from the deck**: the entire value chain of observability is only as strong as the wiring between its layers. You can have perfectly instrumented code, correctly scraped Prometheus metrics, and beautifully designed dashboards — but if the UID reference breaks the datasource-panel binding, the signal never surfaces. The fix took one line of YAML (`uid: prometheus`) and a container restart, but without understanding Grafana's provisioning lifecycle, this would have consumed hours of debugging. In production, I would enforce explicit UIDs in all datasource provisioning configs as a team standard, preventing this class of silent failure from ever reaching staging.

The second-most impactful change was understanding tail-sampling's `decision_wait: 30s` parameter. Before tuning this, forced-error traces were not appearing in Jaeger because the 30-second hold window meant the OTel collector was buffering traces and making sampling decisions based on complete trace data — but the Jaeger query was happening before the decision was committed. Setting an appropriate wait time (or using probabilistic sampling for debugging) is the difference between "tracing looks broken" and "tracing is working correctly but you're querying too early."
