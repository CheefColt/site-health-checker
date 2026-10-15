# Site Reliability & Health Suite (Render Workflows Starter)

A production-grade Site Health Checker template in **TypeScript** and **Python** demonstrating core Render Workflows architectural patterns: **fan-out bounded parallelism**, **conditional dynamic triage**, and **fan-in statistical synthesis**.

**Zero credentials needed:** Only requires standard outbound internet access—no API keys, no domain verification, no third-party accounts.

| Pattern Demonstrated | Where in the Code | Why It Shows Off Render Workflows |
|---|---|---|
| **Bounded Fan-Out** | `probeSite` / `probe_site` | Probes 50+ URLs side-by-side in batches of 10. A sequential loop would take 60+ seconds; in a workflow it takes only as long as the slowest site. |
| **Smart Retries vs Fast Fail** | `probe.ts` / `probe.py` | Transient errors (timeouts, network drops, HTTP 5xx) are retried automatically with exponential backoff. Deterministic 404 client errors complete immediately without wasting runs. |
| **Conditional Branching** | `diagnoseFailure` / `diagnose_failure` | Only failing endpoints trigger deep diagnostic triage (raw DNS resolution, TCP handshake, TLS certificate validation). Healthy sites skip this entirely. |
| **Fan-In Judge / Synthesis** | `generateReport` / `generate_report` | Once all probes and triage tasks settle, the orchestrator passes results to a final judge task that computes SLA uptime %, p95 latency, SSL expiry warnings, and an Executive Markdown Brief. |

---

## 1. Quick Start (Offline Tests, Zero Cost)

You can run the offline test suites without any Render account or network connection.

### TypeScript (Node 20+)
```bash
cd site-health-checker/typescript
npm install
npm test
npm run typecheck
```

### Python (3.10+)
```bash
cd site-health-checker/python
python -m venv .venv
# Windows:
.venv\Scripts\Activate.ps1
# macOS/Linux:
# source .venv/bin/activate
pip install -r requirements.txt pytest pytest-asyncio
pytest -q
```

---

## 2. Architecture & Data Flow

```
                      checkSites (Orchestrator)
                                 │
            ┌────────────────────┴────────────────────┐
            │                                         │
 1. [ probeSite x N ]                      2. [ diagnoseFailure x M ]
    (Parallel Fan-Out)                        (Conditional Fan-Out)
    • HTTP status & latency                   • DNS resolution test
    • SSL certificate inspection              • TCP handshake test
    • Defense-in-depth headers                • TLS handshake test
    • Retries 5xx/timeouts                    • Runs ONLY for failed sites!
            │                                         │
            └────────────────────┬────────────────────┘
                                 │
                    3. generateReport (Fan-In Judge)
                       • Global SLA uptime %
                       • Latency distribution (p50, p95, max)
                       • Security posture grades (A to F)
                       • SSL certificates expiring in <= 30 days
                       • Executive Markdown & JSON output
```

---

## 3. Running Locally with Render CLI

To run the local task server and trigger runs locally:

1. In one terminal, start the task server:
   ```bash
   # TypeScript:
   render workflows dev -- npm start
   # Python:
   render workflows dev -- python main.py
   ```

2. In another terminal, trigger the workflow with sample sites:
   ```bash
   # TypeScript:
   RENDER_USE_LOCAL_DEV=true npm run trigger -- sample/sites.json
   # Python:
   RENDER_USE_LOCAL_DEV=true python trigger.py sample/sites.json
   ```

---

## 4. Deploying to Render

1. Push your repository to GitHub, GitLab, or Bitbucket.
2. In the **Render Dashboard**, click **New > Workflow**.
3. Select your repository.
4. Set **Root Directory** to `site-health-checker/typescript` or `site-health-checker/python`.
5. Configuration:
   - **TypeScript**: Build: `npm install`, Start: `npm start`
   - **Python**: Build: `pip install -r requirements.txt`, Start: `python main.py`
6. Click **Deploy Workflow**.
7. In the Render Dashboard under **Tasks**, start `checkSites` / `check_sites` with a JSON list of URLs:
   ```json
   [["https://github.com", "https://render.com", "https://expired.badssl.com", "https://httpstat.us/404"]]
   ```

---

## 5. Sample Output

```markdown
# 🌐 Site Fleet Reliability Report
> Generated at 2026-10-15 across 16 endpoints.

### 📊 Fleet Status Overview
| Total | 🟢 UP | 🟡 SLOW | 🔴 DOWN | ⚪ BAD (4xx) | Uptime % |
|:---:|:---:|:---:|:---:|:---:|:---:|
| **16** | **12** | **1** | **2** | **1** | **81.3%** |

### ⚡ Latency Profile (Healthy Endpoints)
| Min | Median (p50) | p95 | Max | Average |
|:---:|:---:|:---:|:---:|:---:|
| 84ms | 142ms | **1,850ms** | 2,100ms | 310ms |

### 🚨 Active Incidents & Root-Cause Triage
| Status | Target URL | Failure Stage | Root Cause & Diagnosis | Recommended Action |
|:---:|:---|:---:|:---|:---|
| 🔴 DOWN | `https://expired.badssl.com` | **TLS** | `SSL_CERTIFICATE_EXPIRED`<br><small>certificate has expired</small> | Renew the TLS/SSL certificate immediately with Let’s Encrypt. |
| 🔴 DOWN | `https://this-domain-will-fail-dns-resolution-revhack.org` | **DNS** | `DNS_RESOLUTION_FAILED`<br><small>ENOTFOUND</small> | Check domain registration or DNS nameservers. |
| ⚪ CLIENT | `https://httpstat.us/404` | **HTTP** | `CLIENT_ERROR`<br><small>HTTP 404 Not Found</small> | Verify resource URL endpoint. |
```
