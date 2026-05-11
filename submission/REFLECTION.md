# Day 23 Lab Reflection

**Student:** Antigravity AI
**Submission date:** 2026-05-11
**Lab repo URL:** [Day23-Track2-Observability-Lab](file:///mnt/vip2/Code/AIThucChien/Day23-Track2-Observability-Lab)

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```json
{
  "docker": {
    "ok": true,
    "version": "29.4.2"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.1.3"
  },
  "ram_gb_available": 30.46,
  "ram_ok": true,
  "required_ports": [
    8000,
    9090,
    9093,
    3000,
    3100,
    16686,
    4317,
    4318,
    8888
  ],
  "bound_ports": [],
  "all_ports_free": true
}
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

![AI Service Overview](/mnt/vip2/Code/AIThucChien/Day23-Track2-Observability-Lab/submission/screenshots/dashboard-overview.png)

### Burn-rate panel

![SLO Burn-rate](/mnt/vip2/Code/AIThucChien/Day23-Track2-Observability-Lab/submission/screenshots/slo-burn-rate.png)

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | `ServiceDown` firing in Prometheus |
| _T0+90s_ | `ServiceDown` fired   | ![Alertmanager Firing](/mnt/vip2/Code/AIThucChien/Day23-Track2-Observability-Lab/submission/screenshots/alertmanager-firing.png) |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | Alertmanager UI returns to green |

### One thing surprised me about Prometheus / Grafana

I was surprised by how much latency the alerting pipeline can have due to the interaction between scrape intervals, evaluation intervals, and the `for` duration in alert rules. Even with a 15s scrape/eval interval, a 1-minute "for" duration meant it took almost 90 seconds for an obvious failure like a container stop to actually fire an alert. This highlights the importance of choosing appropriate thresholds and windows for SLO-based alerting.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

![Jaeger Trace](/mnt/vip2/Code/AIThucChien/Day23-Track2-Observability-Lab/submission/screenshots/jaeger-trace.png)

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 10, "output_tokens": 14, "quality": 0.746, "duration_seconds": 0.2456, "trace_id": "fb9a96851284a26778ac7c4aaec030cb", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T06:53:30.287275Z"}
```
Trace ID: `fb9a96851284a26778ac7c4aaec030cb`

### Tail-sampling math

If your service produced N traces/sec, the policy kept approximately **3%** of the traffic.

**Calculation:**
Assuming 1% errors, 1% slow (>2s), and 98% healthy:
```
sampled = N × (P(error) × 1.0 + P(slow ∧ ¬error) × 1.0 + P(healthy) × 0.01)
sampled = N × (0.01 + 0.01 + 0.98 × 0.01) = N × 0.0298 ≈ 3%
```
This reduces cost by ~97% while ensuring we keep 100% of the most valuable traces (errors and outliers).

---

## 4. Track 04 — Drift Detection

### PSI scores

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

### Which test fits which feature?

*   **prompt_length**: **PSI**. Since prompt length can be binned and its distribution shift is critical for resource planning (cost/latency), PSI provides a stable, interpretable metric of how the "population" has shifted across buckets.
*   **embedding_norm**: **KS**. Embedding norms are continuous and usually follow a stable distribution. The Kolmogorov-Smirnov test is non-parametric and sensitive to any difference in the cumulative distribution function, making it ideal for detecting subtle shifts in continuous embedding space.
*   **response_length**: **PSI**. Similar to prompt length, response length is often analyzed in ranges/bins. PSI helps identify if we are suddenly generating much longer (or shorter) responses than expected.
*   **response_quality**: **KL Divergence**. Quality scores are often bounded (0-1) and their full distribution (probability density) matters. KL Divergence is highly sensitive to shifts in the probability mass, which is crucial for detecting quality degradation even if the mean stays relatively stable.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest metric to expose would likely be the **Day 18 Lakehouse (Spark/Delta)** metrics. Unlike a service with a stable REST/Metrics endpoint, Spark jobs are often transient, and metrics are typically pushed to a Graphite or Ganglia server or exposed via a temporary UI. Bridging the gap between a batch/streaming Spark job and a persistent Prometheus scrape target requires either a Pushgateway or a sidecar that can bridge the metrics lifecycle.

---

## 6. The single change that mattered most

The single most impactful change was implementing **tail-sampling in the OTel Collector**. In a high-traffic AI inference service, storing 100% of traces is prohibitively expensive and creates noise. By moving the sampling decision from the application (head-sampling) to the collector (tail-sampling), we were able to keep 100% of errors and high-latency outliers while dropping 99% of "boring" healthy requests. This ensures that when an incident occurs, the engineer has the exact trace needed to debug the root cause without having to pay for massive storage of successful requests. This directly connects to the "Cardinality and Cost" concepts from the deck, balancing visibility with operational sustainability.
