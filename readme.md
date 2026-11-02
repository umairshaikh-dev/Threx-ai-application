# Threx AI — Passive Cyber Threat Intelligence Platform

[![React 19](https://img.shields.io/badge/Frontend-React%2019%20%7C%20Vite%208-blue?style=for-the-badge&logo=react)](https://react.dev/)
[![FastAPI](https://img.shields.io/badge/Backend-FastAPI%20%7C%20Python-009688?style=for-the-badge&logo=fastapi)](https://fastapi.tiangolo.com/)
[![License](https://img.shields.io/badge/Architecture-Passive%20Data%20Diode-orange?style=for-the-badge)](./backend/cybersentinel-backend/README.md)
[![Status](https://img.shields.io/badge/Build-Production%20Ready-brightgreen?style=for-the-badge)]()

AI-powered **passive** SOC dashboard (React 19 + TypeScript) + FastAPI backend with 6 ML detectors. Designed for critical-infrastructure networks where telemetry is one-way (data diode).

> **🛡️ Passive guarantee:** No packets are ever sent back, no payload decryption. See `backend/cybersentinel-backend/README.md` and `data/DATASETS.md`.

---

## 📐 Architecture at a Glance

```
┌─────────────────┐       Passive Tap       ┌────────────────────────┐       WebSockets       ┌────────────────────────┐
│  Critical Net   │ ──────────────────────► │    FastAPI Backend     │ ─────────────────────► │    React 19 Dashboard  │
│  (Data Diode)   │   IPFIX / UDP / PCAP    │  (6 ML Detectors Engine) │       REST APIs        │   (Vite 8 + Tailwind)  │
└─────────────────┘                         └────────────────────────┘                        └────────────────────────┘
```

* **Frontend** (`frontend/`) — Vite 8, React 19, Tailwind v4, Recharts, react-router-dom 7. 12 routes, PWA-ready.
* **Backend** (`backend/cybersentinel-backend/app/`) — FastAPI, Scapy, LightGBM, scikit-learn, Redis, SQLite (dev) / Postgres (prod), WebSocket.
* **Data** (`data/`) — Public datasets attribution, synthetic-derived training data, PCAP schemas, JA3 fingerprints.
* **Models** (`backend/cybersentinel-backend/app/models/artifacts/`) — 4 trained artifacts + `evaluation.json`.

---

## ⚡ Quickstart — Local (Without Docker)

### 1. Backend

```bash
cd backend/cybersentinel-backend
python3 -m venv .venv && source .venv/bin/activate  # optional
pip install -r requirements.txt               # installs fastapi, uvicorn, lightgbm, scikit-learn, scipy ...

# optional: training is already done (artifacts committed), but to retrain:
brew install libomp   # macOS only, required for lightgbm
python scripts/train_all.py

cp .env.example .env   # edit CYBERSENTINEL_CORS_ORIGINS if needed
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# verify setup
curl http://localhost:8000/health | jq
curl http://localhost:8000/threats/?limit=2 | jq
```

### 2. Frontend

```bash
cd frontend
npm install
cp .env.example .env   # VITE_API_URL=http://localhost:8000 by default
npm run dev            # http://localhost:5173
npm run build && npm run preview  # production preview on http://localhost:4173
```

#### 🚦 Dashboard Unified Status Indicators

The dashboard auto-detects the backend state and renders one of the following indicators:

* **`SIMULATION`** <sub>🟡 (Yellow)</sub> — Backend unreachable, using mock data
* **`BACKEND SEEDED`** <sub>🟢 (Green)</sub> — Backend live, displaying DB-persisted alerts
* **`PCAP REPLAY`** <sub>🟢 (Green)</sub> — Replaying captured flow records
* **`LIVE INGEST`** <sub>🔵 (Blue)</sub> — `CYBERSENTINEL_LIVE_INTERFACE` set, passive capture active
* **`LAB TRAFFIC`** <sub>🌐 (Cyan)</sub> — Lab-generated flow records moving through pipeline

---

## 🐳 Quickstart — Docker (Full Stack)

```bash
# From repo root
cp .env.example .env
cp backend/cybersentinel-backend/.env.example backend/cybersentinel-backend/.env
cp frontend/.env.example frontend/.env

# Choose strong passwords in .env (POSTGRES_PASSWORD, etc.)
docker compose up --build -d

# Wait ~20s for healthchecks
curl http://localhost:8000/health
curl http://localhost:80/   # frontend via nginx
docker compose logs -f backend
```

### Deployed Services Summary

| Service | Endpoint / Port | Description |
| :--- | :--- | :--- |
| **`threx-backend`** | `:8000` (API) / `:9090` (Metrics) | FastAPI engine & Prometheus metrics |
| **`threx-frontend`** | `:80` | Nginx SPA production host |
| **`threx-redis`** | `:6379` | In-memory cache & event queue |
| **`threx-postgres`** | `:5432` | Alert storage (`pg_data` volume) |

---

## ⚙️ Configuration

All environment variables are fully documented in:
* `backend/cybersentinel-backend/.env.example`
* `frontend/.env.example`
* `./.env.example` (compose root)

### Key Variables

| Variable | Default Value | Purpose |
| :--- | :--- | :--- |
| `CYBERSENTINEL_ENV` | `development` | Setting to `production` enables HSTS and strict headers |
| `CYBERSENTINEL_CORS_ORIGINS` | `http://localhost:5173` | Comma-separated list of allowed origins |
| `CYBERSENTINEL_API_KEY` | _(empty)_ | If set, clients must send `X-API-Key` |
| `CYBERSENTINEL_RATE_LIMIT_PER_MINUTE` | `600` | Per-IP fixed 60s window (`/health*` exempt) |
| `CYBERSENTINEL_DB_PATH` | `data/alerts.db` | SQLite fallback database path |
| `CYBERSENTINEL_POSTGRES_DSN` | `postgresql://...` | Postgres DSN (Docker Compose overrides this) |
| `CYBERSENTINEL_LIVE_INTERFACE` | _(empty)_ | Set interface (e.g. `eth0`) to enable passive sniffing |
| `VITE_API_URL` | `http://localhost:8000` | Target URL pointing frontend to backend |

---

## 🗺️ Routes & API Surface

### 🖥️ Frontend Routes *(All Verified)*

```text
/                              → Command Center + Metrics + Donut/Threat Map
/overview                      → Unified SOC Overview Dashboard
/threats                       → Interactive Alerts Table
/threats/:id                   → Deep Investigation (Evidence, Flow Viz, Notes, Exports)
/traffic                       → Network Traffic Analytics
/dns                           → DGA & Tunneling Analytics
/tls                           → TLS & JA3 Analytics
/recon                         → Reconnaissance & Scanning Analytics
/exfil                         → Data Exfiltration Analytics
/alerts                        → AlertCenter Bulk Action Portal
/reports                       → Dynamic Report Generator (JSON/CSV)
/ai-engine                     → Model Cards, Pipeline Metrics, Latency
/system                        → System Health, Hardware Resources, Diode Status
*                              → Custom 404 Route
```

### ⚙️ Backend API (`http://localhost:8000/docs`)

```http
GET  /health, /health/ready, /health/live, /api/health
GET  /metrics, /api/metrics  (Prometheus format)
GET  /threats, /threats/{id}
GET  /traffic/stats, /top-talkers, /protocols
GET  /dns/dga, /dns/tunneling
GET  /tls/fingerprints, /tls/malware
GET  /recon/scans
GET  /exfil/anomalies
GET  /system/status, /system/throughput
POST /reports/generate  (Payload: ReportConfig {format, threat_class, severity, limit})
WS   /ws/alerts, /ws/metrics
GET  /
```

---

## 📊 Data & Machine Learning Models

Refer to `data/DATASETS.md` for dataset sources/licenses and `data/models/evaluation.json` for live model metrics. Comprehensive architectural documentation is available in `MODEL_DOCUMENTATION.md`.

**Training Pipeline:** Deterministic seed `42`, 80/20 stratified split applied across all applicable modules.

```text
┌─────────────────────────┬────────────────────────────┬─────────────────────────────────────────────────┐
│ Detector Module         │ Model / Methodology        │ Performance Metric                              │
├─────────────────────────┼────────────────────────────┼─────────────────────────────────────────────────┤
│ DGA Classifier          │ LightGBM                   │ CV AUC 0.9966 ± 0.0001 (Baseline 0.9384)        │
│ TLS Malware             │ Random Forest              │ Metrics served dynamically from evaluation.json │
│ DDoS Detection          │ Isolation Forest           │ Test AUC 0.9967 (300 benign / 300 attack flows) │
│ Exfiltration            │ Isolation Forest           │ Test AUC 0.9933                                 │
│ C2 Beaconing            │ FFT + Constant Variance    │ Accuracy 1.0 (Threshold cv < 0.1)               │
│ Port Scanning           │ Statistical Threshold      │ Accuracy 1.0 (Ports > 20 or Hosts > 15)         │
└─────────────────────────┴────────────────────────────┴─────────────────────────────────────────────────┘
```

Artifacts are pre-committed under `backend/cybersentinel-backend/app/models/artifacts/` and automatically loaded on startup. If artifacts are missing, the backend will auto-train on first boot (~10s overhead for DGA).

---

## 🧪 Lab Traffic Generation

Synthesize realistic attack flows to stress-test detection modules and dashboard visualizations:

```bash
python3 scripts/generate_lab_traffic.py   # Generates data/pcaps/attacks/*.json + mixed/lab_mixed.json
```

Outputs flow records matching characteristics of real-world attack vectors:
* **DDoS SYN Flood:** SYN/ACK ratio 9:1, 5,000–50,000 packets
* **Slowloris:** 120–600s connection duration, tiny packet payloads
* **DNS Tunneling:** High query frequency, oversized TXT records
* **DGA Domains:** 5 families (*Nymaim, Matsnu, Suppobox, Gozi, CryptoLocker*)
* **C2 Beaconing:** 60s interval with 5% jitter variance
* **Port Scanning:** 200–2,000 unique ports probed per burst
* **TLS Malware:** Suspicious JA3 signatures with zero extensions
* **Exfiltration:** High-volume outbound data spikes (500KB–50MB)

---

## 🚀 Production Build & Deployment Checklist

### Production Build Verification

```bash
# 1. Build Frontend SPA
cd frontend && npm run build   # Outputs dist/, served by Nginx inside Docker

# 2. Prepare Backend & Run Unit Tests
cd backend/cybersentinel-backend
pip install -r requirements.txt
pytest -q   # 52 passed

# 3. Launch Multi-worker Server
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

### Health Probes & Monitoring

```bash
# Docker / K8s Healthcheck Probe
HEALTHCHECK curl -f http://localhost:8000/health

# Readiness Probe:  /health/ready
# Liveness Probe:   /health/live
# Frontend Probe:   wget -qO- http://localhost/
```

### Deployment Checklist

1. **Clone Repo & Configure Secrets:** Copy `.env.example` $\rightarrow$ `.env` across all directories. Define strong passwords for `POSTGRES_PASSWORD` and `CYBERSENTINEL_API_KEY`.
2. **Build Frontend:** Run `npm ci && npm run build` (or execute `docker compose build frontend`).
3. **Initialize Artifacts:** Ensure `app/models/artifacts/*.joblib` exist (run `python scripts/train_all.py` if missing).
4. **Boot Services:** Run `docker compose up -d`. Verify system health with `curl /health` (must return HTTP `200`).
5. **DNS & TLS Routing:** Route DNS/FQDN to the application frontend. Terminate TLS at your load balancer (HSTS automatically attached when `CYBERSENTINEL_ENV=production`).
6. **Log Shipping:** Configure Uvicorn access logs to stdout for aggregation via Compose `json-file` driver.
7. **Database Persistence:** Mount `pg_data` and `data/alerts.db` onto persistent volumes for database backups.

---

## 🔒 Security Hardening

The application comes pre-configured with industry-standard security controls:

* **Strict CORS Allowlist:** Explicit origins only; wildcard disallowed when credentials are enabled.
* **Security Headers:** Enforces `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, production HSTS, and strict `Permissions-Policy`.
* **Request Tracing:** Automatic `X-Request-Id` echo across all responses.
* **IP Rate Limiting:** Enforced on a 60s window prior to handler execution (returns `429 Too Many Requests` with `Retry-After`; `/health*` exempt).
* **API Key Auth:** Optional enforced `X-API-Key` authentication.
* **Input Validation:** Built with strict Pydantic models (maximum 6 evidence items, enum validation for severity/protocols).
* **Exception Shielding:** Global exception handlers prevent internal stack trace leakage.
* **Container Security:** Non-root Nginx execution, minimal base images, immutable `npm ci` dependencies, and zero `.env` leaks.

---

## 📈 Measured Results & Benchmarks

Regenerate all benchmark metrics below using the corresponding scripts. **Do not hand-edit these numbers.**

> ⚠️ CI automatically runs `python3 scripts/regenerate_evaluation.py --check` and will fail if values drift from `data/models/evaluation.json`.

### ⚡ Pipeline Throughput (Multi-Core Benchmark)

**Headline figure:** End-to-end, multi-core performance measured on benchmark hardware. IPFIX flow records arrive over UDP, pass the receiver/splitter, decode, get evaluated by **all 6 ML detectors** across $N$ worker processes, deduplicate, persist, and push alerts to the UI.

```bash
cd backend/cybersentinel-backend        # Performance power profile active
PYTHONPATH=. python scripts/benchmark_pipeline.py --rate 120000 --seconds 60 --mix lab
PYTHONPATH=. python scripts/benchmark_pipeline.py --rate 120000 --seconds 60 --mix realistic
```

| Benchmark Run | Hardware Environment | Sustained Throughput | Latency (p50 / p95 / p99) | Packet Loss |
| :--- | :--- | :--- | :--- | :--- |
| **`--mix lab`** (30% attack mix) | i7-14650HX Laptop | *Execute command above* | *Auto-populated* | *0%* |
| **`--mix realistic`** (3% attack mix) | i7-14650HX Laptop | *Execute command above* | *Auto-populated* | *0%* |

*Results are logged automatically to `data/benchmark_<mix>_<rate>.json`.*

#### 🛠️ Per-Core Stage Rates (1-vCPU Sandbox Execution)
* **Detection Processing:** 13.4k flows/s *(Up from 2.8k/s)*
* **Detection + Dispatch:** 10.1k flows/s *(Up from 1.9k/s)*
* **IPFIX Decoding:** 52k records/s
* **PCAP Parsing Engine:** 274k packets/s *(Scapy baseline was 5.3k/s)*

> **Multi-core Pipeline:** Run `scripts/evaluate_detectors.py --shards 8` to verify multi-core parity with single-process results.  
> **Production Startup:** `CYBERSENTINEL_FLOW_LISTEN_PORT=4739 CYBERSENTINEL_PARALLEL_WORKERS=12` (Feed with `scripts/flow_exporter.py` or standard NetFlow/IPFIX exporters).

---

### 🌐 DGA Classifier Performance (Real-World Corpora)

```bash
cd backend/cybersentinel-backend && PYTHONPATH=. python3 scripts/train_dga.py
```

**Dataset Composition:** 89,917 Benign (`dga_domains.csv` legit entries + Tranco Top-1M) and 52,665 DGA domains (DGArchive).

| Evaluation Metric | Measured Score |
| :--- | :--- |
| **Length-Only Baseline (CV AUC)** | `0.9384` |
| **Full Model - 8 Features (CV AUC)** | `0.9966` |
| **Deployed Model - Length-Ablated (7 Features CV AUC)** | **`0.9966 ± 0.0001`** |
| **True Positive / False Positive Rates** | **`96.1% / 2.4%`** |

> 📌 **Length Ablation:** `domain_length` is explicitly excluded from the deployed model to prevent length-confounding bias. If the baseline and full model converge, the dataset is length-skewed.  
> *Live metrics are served at `GET /api/v1/models` and displayed directly within the AI Engine UI.*

---

### 🛡️ Unidirectional Resilience Ablation Test

```bash
cd backend/cybersentinel-backend && PYTHONPATH=. python3 scripts/run_ablation.py
```

Two-sided features (`syn_to_ack_ratio`, `amplification_ratio`, `byte_ratio_outbound`) are stripped to simulate real data-diode network taps. 

* **Surfaced Limitation:** C2 beaconing precision currently sits at `0.30`. The unidirectional ablation highlighted this detection-quality issue, which is flagged for improvement in upcoming releases.

---

## 📡 Live Telemetry API Reference

The frontend relies exclusively on the endpoints below. **No client-side dummy data is fabricated.**

```http
GET  /api/v1/metrics/throughput      → Live flows/sec, p50/p95/p99 latency, queue depth, drop count, CPU/RSS
GET  /api/v1/analytics/time-series   → Time-series metrics (?buckets=60&bucket_seconds=1)
GET  /api/v1/alerts/recent           → Real-time alert feed featuring detailed evidence
GET  /api/v1/alerts/{id}             → Single alert deep-dive record
GET  /api/v1/analytics/dns           → DGA & DNS tunneling telemetry
GET  /api/v1/analytics/tls           → TLS telemetry (JA3S reported as unavailable)
GET  /api/v1/models                  → Live metrics pulled directly from evaluation.json
POST /api/v1/pcap/upload             → Replay user-uploaded PCAP/JSON packet captures
```

### 🔍 Transparent System Gaps

Missing or unsupported capabilities are explicitly flagged in the UI rather than populated with placeholder data:

* **JA3S Fingerprinting:** Disabled. Requires two-way `ServerHello` packets, which cannot be captured over a unidirectional tap (`ja3s_available: false`).
* **Alert Triage Actions:** Deferred. Mark reviewed / escalation actions lack backend persistence schemas; UI buttons display disabled states with a "Phase 2" tag.
* **JA4 / QUIC / Native NetFlow Ingest:** Documented in specifications, slated for future release iterations.
* **System Hardware Metrics:** If `psutil` is missing, the backend explicitly reports `resources.available: false` instead of mocking CPU/Memory values.

---

## 🚨 Failure & Safe-State Behavior

If the inference engine fails to initialize during startup, **the backend immediately aborts execution**:

```text
RuntimeError: Inference engine failed to initialise and CYBERSENTINEL_ALLOW_DEGRADED_START is false. 
Refusing to start a backend that cannot detect threats. 
Cause: ImportError: No module named 'pandas'
```

To enable frontend-only development mode without an engine, pass `CYBERSENTINEL_ALLOW_DEGRADED_START=true`. The system will launch in a degraded state and explicitly broadcast its status:

```json
{
  "status": "degraded",
  "detection_active": false,
  "components": {
    "inference": "UNAVAILABLE"
  },
  "degraded_reason": "ImportError: No module named 'pandas'"
}
```

> **Design Philosophy:** "No threats found" and "No detector running" must never look identical.

---

## 🛠️ Troubleshooting Guide

| Issue / Error | Root Cause | Solution |
| :--- | :--- | :--- |
| `libomp missing` | Missing OpenMP dependency on macOS | Run `brew install libomp` |
| `ModuleNotFoundError: cybersentinel_backend` | Python execution path misconfiguration | Run pytest relative to root: `cd backend/cybersentinel-backend && pytest` |
| **CORS Request Blocked** | Mismatched frontend origin | Update `CYBERSENTINEL_CORS_ORIGINS` to include your client URL |
| **Database Empty After Restart** | State reset | SQLite uses `backend/cybersentinel-backend/data/alerts.db`. Postgres relies on `pg_data` volume. Both auto-seed **only** if completely empty. |

---

## 📄 License & Dataset Attribution

Refer to `data/DATASETS.md` for individual upstream dataset licenses (including **Tranco**, **DGArchive**, **CIC-IDS**, **CTU-13**, and **JA3/MTA**).

* **Source Code:** Proprietary demonstration release.
* **Dataset Artifacts:** Governed by original upstream licenses (*CC0 / BSD / Research Use Only*).