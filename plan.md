# Plan Hoàn Thành Day 23 — Observability Stack Lab

> AICB · Phase 2 · Track 2 · Day 23

---

## Phase 0: Chuẩn Bị (15-20 phút)

### 1. Kiểm tra phần cứng (theo `HARDWARE-GUIDE.md`)
- [ ] Đảm bảo có >= 4GB RAM free (khuyến nghị 8GB)
- [ ] Docker Desktop: tăng memory limit len >= 6GB (Settings -> Resources)
- [ ] Kiểm tra `docker compose version`
- [ ] Linux: `groups | grep docker` hoặc `sudo usermod -aG docker $USER && newgrp docker`

### 2. Clone & Setup
```bash
git clone <repo> && cd Day23-Track2-Observability-Lab
cp .env.example .env
# Edit .env: them SLACK_WEBHOOK_URL (tao tai https://api.slack.com/messaging/webhooks)
make setup          # Pull images + verify-docker.py
```

### 3. Chon persona (`VIBE-CODING.md`)
- [ ] **SRE**: Tap trung alerts, SLO, incident response
- [ ] **Platform**: Tap trung dashboards, integration, tooling
- [ ] **Data**: Tap trung drift detection, metrics quality

---

## Phase 1: 00-Setup (15 phút) — 5 diem

**Muc tieu**: `setup-report.json` committed

```bash
python3 verify-docker.py
# Commit file 00-setup/setup-report.json (hoac submission/setup-report.json)
```

### Checklist:
- [ ] `setup-report.json` tao thanh cong
- [ ] File duoc commit

---

## Phase 2: 01-Instrument FastAPI (30 phút) — 20 diem

**Muc tieu**: `/metrics` expose du 6 metric families

### 1. Khoi dong stack & kiem tra metrics
```bash
make up
# Doi ~30s cho stack khoi dong
curl http://localhost:8000/metrics | grep -E "inference_requests_total|inference_latency_seconds_bucket|inference_active_gauge|gpu_utilization_percent|inference_tokens_total|inference_quality_score"
```

### 2. Cac metrics can co:
- [ ] `inference_requests_total{model,status}` — Counter
- [ ] `inference_latency_seconds_bucket{model}` — Histogram
- [ ] `inference_active_gauge` — Gauge (tang khi load, ve 0 sau)
- [ ] `gpu_utilization_percent` — Gauge
- [ ] `inference_tokens_total{model,direction}` — Counter
- [ ] `inference_quality_score{model}` — Gauge

### 3. Verify traces & logs
- [ ] Truy cap Jaeger UI: http://localhost:16686
- [ ] Tim trace cho `POST /predict` (co 3 child spans: embed-text -> vector-search -> generate-tokens)
- [ ] Grafana -> Explore -> Loki: tim log co `trace_id`

### Screenshot can chup:
- [ ] Metrics output tu `/metrics`
- [ ] Jaeger trace flame graph
- [ ] Loki log line voi `trace_id`

---

## Phase 3: 02-Prometheus & Grafana (45 phút) — 25 diem

**Muc tieu**: 3 dashboards + alerts -> Slack

### 1. Verify dashboards (http://localhost:3000, admin/admin)
- [ ] `ai-service-overview.json` — 6 panels (RPS, P50-95-99 latency, errors, GPU, tokens, cost)
- [ ] `slo-burn-rate.json` — error budget + burn rate
- [ ] `cost-and-tokens.json` — token throughput + cost estimate

### 2. Generate load
```bash
make load    # 60s locust load, 10 concurrent users
```
- [ ] Dashboards co data khong-zero
- [ ] Chup screenshot moi dashboard

### 3. Test alerts
```bash
make alert
```
- [ ] Giet app container -> doi `ServiceDown` fire (can ~90s)
- [ ] Kiem tra Alertmanager: http://localhost:9093
- [ ] Kiem tra Slack nhan duoc message "fire"
- [ ] App tu restore -> doi message "resolve" tren Slack

### Screenshot can chup:
- [ ] 3 dashboards voi data
- [ ] Alertmanager UI showing fired alerts
- [ ] Slack messages (fire + resolve)

---

## Phase 4: 03-Tracing & Logs (30 phút) — 20 diem

**Muc tieu**: End-to-end trace + tail-sampling verified

### 1. Hieu tail-sampling policy
- Keep 100% errors
- Keep 100% slow (>2s)
- Keep 1% healthy
- Xem `03-tracing-and-logs/otel-collector/sampling-policies.md` de hieu math

### 2. Test tail-sampling
```bash
# Gay loi (kill app) va kiem tra error trace duoc giu lai
make alert  # hoac tu kill container
```
- [ ] Jaeger UI: confirm error traces retained
- [ ] Healthy traces: chi ~1% duoc giu

### 3. Log-to-trace correlation
- [ ] Grafana -> Explore -> Loki
- [ ] Tim log line co `trace_id`
- [ ] Click `trace_id` -> jump sang Jaeger trace

### Screenshot can chup:
- [ ] Jaeger trace cho `POST /predict` voi 3 child spans
- [ ] Span attributes panel (GenAI semantic conventions)
- [ ] Loki log line + trace correlation

---

## Phase 5: 04-Drift Detection (20 phút) — 15 diem

**Muc tieu**: `drift-summary.json` + Evidently report

```bash
cd 04-drift-detection
python3 scripts/drift_detect.py
# Generates:
#   reports/drift-report.html
#   reports/drift-summary.json
```

### 1. Verify
- [ ] `drift-summary.json` co >=1 feature voi `drift: yes` va PSI > 0.2
- [ ] Mo `reports/drift-report.html` trong browser

### 2. Viet REFLECTION (phan nay)
- [ ] Giai thich khi nao dung PSI vs KL vs KS vs MMD:
  - PSI: categorical / binned continuous
  - KL: distribution similarity (can smooth)
  - KS: continuous, non-parametric
  - MMD: high-dimensional, non-linear

### Screenshot can chup:
- [ ] Evidently HTML report
- [ ] `drift-summary.json` content

---

## Phase 6: 05-Integration (20 phút) — 15 diem

**Muc tieu**: Cross-day dashboard voi >=1 source

### 1. Neu co prior days running:
- [ ] Uncomment targets trong `prometheus.yml`
- [ ] Set `DAY19_QDRANT_URL` va `DAY20_LLAMACPP_METRICS_URL` trong `.env`
- [ ] `make restart`

### 2. Neu khong co:
- [ ] Chay stub scripts trong `05-integration/`
- [ ] Dashboard van render voi "No Data" cho cac panel khong co source

### 3. Verify
- [ ] Dashboard `full-stack-dashboard.json` co 6 panels
- [ ] It nhat 1 panel co data (hoac stub running)

### Screenshot can chup:
- [ ] Cross-day dashboard

---

## Phase 7: Reflection & Submission (20 phút) — 15 diem

### 1. Viet `submission/REFLECTION.md` voi 5 sections:
- [ ] **Section 1**: Tom tat stack da xay
- [ ] **Section 2**: Kho khan gap phai & cach giai quyet
- [ ] **Section 3**: Tail-sampling math (tinh toan retention rate ~3%)
- [ ] **Section 4**: Structured log line voi `trace_id` (paste vi du)
- [ ] **Section 5**: **"The single change that mattered most"** (doan quan trong nhat)

### 2. Thu thap screenshots vao `submission/screenshots/`:
- [ ] Setup report
- [ ] Metrics (6 families)
- [ ] 3 dashboards
- [ ] Alerts (fire + resolve)
- [ ] Slack messages
- [ ] Jaeger trace
- [ ] Loki log + trace correlation
- [ ] Drift report
- [ ] Cross-day dashboard

### 3. Final verify:
```bash
make verify    # Exit 0 = ready to submit
```

---

## Bonus (Optional, +20 diem)

| Bonus | Thoi gian | Yeu cau |
|-------|-----------|---------|
| **B1: eBPF Profiling** | +30 phút | Linux/WSL only. Pyroscope flame graph cho `day23-app` |
| **B2: LLM Native Observability** | +30 phút | Self-hosted Langfuse, capture 1 LangChain trace |

---

## Tong thoi gian uoc tinh

| Phase | Thoi gian |
|-------|-----------|
| Core tracks (00-05) | ~2 gio |
| Reflection & submission | ~20 phút |
| **Bonus** | **+1 gio** |

---

## Tips quan trong

1. **Slack webhook**: Test truoc bang `curl -X POST $SLACK_WEBHOOK_URL -d '{"text":"test"}'`
2. **Grafana provisioning**: Neu dashboards khong hien, doi 30s roi refresh
3. **Alert dwell time**: `make alert` can patience — alert fire sau ~90s
4. **Memory**: Neu RAM < 4GB, skip bonus tracks va giam locust concurrency
5. **Screenshot**: Chup ngay sau moi step, dung de cuoi cung

---

## Rubric Mapping

| # | Track | Diem | Checkpoint | Trang thai |
|---|-------|------|------------|------------|
| 1 | 00-setup | 5 | `setup-report.json` committed | [ ] |
| 2 | 01 instrument | 5 | `/metrics` exposes `inference_requests_total` | [ ] |
| 3 | 01 instrument | 5 | `/metrics` exposes `inference_latency_seconds_bucket` | [ ] |
| 4 | 01 instrument | 5 | `/metrics` exposes `inference_active_gauge` | [ ] |
| 5 | 01 instrument | 5 | `inference_quality_score` va `inference_tokens_total` | [ ] |
| 6 | 02 dashboards | 5 | 3 dashboards loaded automatically | [ ] |
| 7 | 02 dashboards | 5 | Overview dashboard 6 panels render | [ ] |
| 8 | 02 dashboards | 5 | SLO burn-rate dashboard populates | [ ] |
| 9 | 02 dashboards | 5 | Cost-and-tokens dashboard non-zero $/hr | [ ] |
| 10 | 02 alerts | 5 | `make alert` triggers `ServiceDown` | [ ] |
| 11 | 02 alerts | 5 | Slack receives fire AND resolve | [ ] |
| 12 | 03 tracing | 5 | Jaeger UI trace `POST /predict` w/ 3 child spans | [ ] |
| 13 | 03 tracing | 5 | Span attributes follow GenAI conventions | [ ] |
| 14 | 03 tracing | 5 | tail-sampling: error retained, healthy dropped | [ ] |
| 15 | 03 logs | 5 | Structured JSON log voi `trace_id` | [ ] |
| 16 | 04 drift | 5 | `drift-summary.json` exists & drift: yes | [ ] |
| 17 | 04 drift | 5 | Evidently HTML report renders | [ ] |
| 18 | 04 drift | 5 | REFLECTION explains PSI/KL/KS/MMD | [ ] |
| 19 | 05 integ | 5 | >=1 prior-day source connected | [ ] |
| 20 | 05 integ | 5 | Cross-day dashboard 6 panels | [ ] |
| 21 | reflection | 5 | REFLECTION.md sections 1-5 filled | [ ] |
| 22 | reflection | 10 | "The single change that mattered most" | [ ] |
| B1 | eBPF bonus | +10 | Pyroscope flame graph | [ ] |
| B2 | Langfuse bonus | +10 | LangChain trace captured | [ ] |
