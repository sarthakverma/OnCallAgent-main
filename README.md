# OnCallAgent: AI-Powered On-Call System

An intelligent monitoring and alerting system that uses AI to automatically diagnose issues, fetch logs, and notify on-call engineers via WhatsApp and Jira integration.

## Quick Start

Get the system running in minutes:

1. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

2. **Start the core services:**
   ```bash
   # Terminal 1: Process Manager (orchestrates inventory services)
   python process_manager.py

   # Terminal 2: Scale up inventory instances
   curl -X POST localhost:7070/scale -H "Content-Type: application/json" -d '{"replicas":2}'

   # Terminal 3: Load Balancer (routes traffic to inventory services)
   uvicorn loadbalancer:app --port 9000

   # Terminal 4: Prometheus (monitoring and alerting)
   prometheus --config.file=prometheus.yaml

   # Terminal 5: Alertmanager (forwards alerts to AI agent)
   alertmanager --config.file=alertmanager.yaml

   # Terminal 6: AI Agent (handles alerts and notifications)
   uvicorn on_call_agent:app --port 8088
   ```

3. **Generate traffic to trigger alerts:**
   ```bash
   python traffic_gen.py --url http://localhost:9000 --profile agentic --duration 180
   ```

4. **Set up notifications (optional):**
   - Add `OPENAI_API_KEY` to `.env` for AI diagnosis
   - Configure Twilio credentials in `.env` for WhatsApp notifications
   - Set up Jira credentials in `.env` for ticket creation

**Default Ports:**
- Process Manager: 7070
- Inventory Services: 8001-8010
- Load Balancer: 9000
- Prometheus: 9090
- Alertmanager: 9093
- AI Agent: 8088

## System Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Traffic Gen    │───▶│  Load Balancer  │───▶│  Inventory Svc  │
│                 │    │     (9000)      │    │   (8001-8010)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                       │
                                ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Process Manager │◀───│    Prometheus   │◀───│   Metrics       │
│    (7070)       │    │   (9090)        │    │   Collection    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Service Discov. │    │   Alert Rules   │    │   Alertmanager  │
│  (targets.json) │    │  (alerts.yaml)  │    │    (9093)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                       │
                                ▼                       ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │  AI On-Call     │◀───│   Alert         │
                       │  Agent (8088)   │    │  Webhook        │
                       └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │  WhatsApp/Jira  │
                       │ Notifications    │
                       └─────────────────┘
```

## Glossary

- **Service Discovery**: Automatic detection of service instances for monitoring
- **Round-Robin Load Balancing**: Distributes requests evenly across available services
- **Prometheus Metrics**: Standardized measurements (counters, histograms, gauges) for monitoring
- **Alert Rules**: Conditions that trigger when metrics exceed thresholds
- **Instant Query**: Real-time metric queries in Prometheus (vs. range queries over time)
- **File-based Service Discovery**: Using JSON files to tell Prometheus which services to monitor

---



Below is a complete, repo-grounded sequence for bringing the system up, with each component’s role, the important parts of its implementation, and why it must be started in this order. File references are to files in the repo.


Components at a glance
* Inventory service (inventory_service.py): FastAPI app exposing CRUD-like endpoints and simulated failure endpoints. Exposes Prometheus metrics.
* Process manager (process_manager.py): Starts/stops/scales inventory service instances, writes Prometheus file-based SD target list.
* Load balancer (loadbalancer.py): Simple reverse proxy with round-robin + failover; periodically fetches current backends from the process manager.
* Prometheus (prometheus.yaml): Scrapes metrics from inventory instances via file SD and loads alert rules.
* Alertmanager (alertmanager.yaml): Sends alert webhooks to the on-call agent.
* On-Call AI agent (on_call_agent.py): Receives alerts, calls OpenAI functions to fetch logs and error rate, diagnoses, and optionally notifies via WhatsApp and creates Jira tasks.
* Traffic generator (traffic_gen.py): Drives realistic/agentic traffic to the load balancer to exercise success and error paths.
* Rules (alerts.yaml): Alert when inventory error rate > 20% over 1m.
* Targets file (inventory_targets.json): File SD target list maintained by the process manager.


## Detailed Setup Guide

### Phase 1: Preparation
**Install dependencies**
* Significance: Ensures required libs exist to run all services.
* Important parts: `requirements.txt` includes `fastapi`, `uvicorn`, `httpx`, `openai`, `prometheus-client`, `twilio`, `python-dotenv`.
* Why now: All subsequent Python components depend on these.

### Phase 2: Core Infrastructure
**Start the Process Manager**
* Command: `python process_manager.py`
* Port: 7070
* Role: Orchestrates inventory service instances and produces the Prometheus service discovery file `inventory_targets.json`
* Key Features:
  - `spawn_instance()` and `wait_healthy()` start/verify each `inventory_service.py` instance
  - `@app.post("/start")`, `@app.post("/scale")`, `@app.get("/instances")` to manage replicas
  - `write_sd_file()` and `_sd_payload()` write `inventory_targets.json` for Prometheus
  - `@app.get("/backends")` returns live backend URLs used by the load balancer
  - Inventory instances default ports: 8001-8010
* Dependencies: Must start first as Prometheus and Load Balancer depend on its targets file

**Scale up inventory instances**
* Command: `curl -X POST localhost:7070/scale -H "Content-Type: application/json" -d '{"replicas":2}'`
* Role: Ensures multiple instances exist for the load balancer and Prometheus to target
* Key Features:
  - Instances run `inventory_service.py` with unique `SERVICE` names and ports
  - Exposes: `/items/{id}` (GET/PUT), `/random-endpoint` (simulated errors), `/metrics`, `/logs/tail`, `/faults`
  - Metrics: `http_requests_total`, `http_errors_total`, `http_request_duration_seconds`, `inventory_items`
  - Central error log: `logs/inventory_common.log` (ERROR level only)
* Dependencies: Backends must exist before starting Load Balancer and Prometheus

### Phase 3: Load Balancing
**Start the Load Balancer**
* Command: `uvicorn loadbalancer:app --port 9000`
* Port: 9000
* Role: Front-door endpoint; fans out traffic to available backends via round-robin with failover
* Key Features:
  - Environment: `PM_URL` (default `http://127.0.0.1:7070`) to fetch dynamic backends
  - Refresh interval: `LB_REFRESH_SEC` (default 5s)
  - `@app.api_route("/{path:path}")` proxies all methods with failover logic
  - `pick_primary_and_alt()` implements round-robin with backup selection
  - `GET /healthz` shows current dynamic backends in use
* Dependencies: Must be up before traffic generation; AI agent uses it to fetch logs

### Phase 4: Monitoring & Alerting
**Start Prometheus**
* Command: `prometheus --config.file=prometheus.yaml`
* Port: 9090
* Role: Scrapes inventory metrics and evaluates alert rules
* Key Features:
  - `scrape_configs.file_sd_configs.files` watches `./inventory_targets.json` from Process Manager
  - `rule_files`: `./alerts.yaml` loads the error rate alert rule
  - `alerting.alertmanagers.targets`: `["localhost:9093"]` connects to Alertmanager
  - Global scrape interval: 5s
* Dependencies: Must run before alerts can fire; relies on Process Manager's targets file

**Start Alertmanager**
* Command: `alertmanager --config.file=alertmanager.yaml`
* Port: 9093
* Role: Receives alert notifications from Prometheus and forwards them to the on-call agent webhook
* Key Features:
  - `receivers.webhook_configs.url`: `"http://127.0.0.1:8088/alerts"` points to `on_call_agent.py`
  - Grouping, waits, and intervals for batching alerts
* Dependencies: Must exist to receive Prometheus alerts; AI agent endpoint must be up

### Phase 5: AI Agent
**Start the On-Call AI Agent**
* Command: `uvicorn on_call_agent:app --port 8088`
* Port: 8088
* Role: Central brain that handles alerts, fetches logs and error rates, diagnoses, notifies via WhatsApp, and optionally creates Jira tasks
* Environment Variables:
  - `AGENT_PORT=8088`, `LB_URL=http://127.0.0.1:9000`, `PROM_URL=http://127.0.0.1:9090`
  - Optional: `OPENAI_API_KEY`, `GITHUB_TOKEN`, Twilio (`TWILIO_*`), Jira (`JIRA_*`)
* Endpoints:
  - `POST /alerts`: Main alert webhook called by Alertmanager
  - `POST /webhook/twilio`: Handles WhatsApp user replies (Twilio webhook)
  - `GET /healthz`
* Alert Workflow:
  - LLM conversation with `SYSTEM_PROMPT` for parallel function calls
  - `fetch_logs()` → `fetch_recent_logs()` pulls from `LB_URL/logs/tail`
  - `compute_error_rate()` queries Prometheus instant vectors via `prom_instant_query()`
  - May call `fetch_code_from_github()` for pinpointing code lines
  - Concludes with `send_diagnosis` and formats WhatsApp message via `send_whatsapp_message()`
* Integrations:
  - WhatsApp: Optional via Twilio (logs and skips if creds missing)
  - Jira: Created via `create_jira_issue()` if user replies with instruction
* Dependencies: Must be running before Alertmanager delivers webhooks

### Phase 6: Traffic Generation
**Drive traffic with the Traffic Generator**
* Command: `python traffic_gen.py --url http://localhost:9000 --profile agentic --duration 180`
* Role: Produces both normal and error traffic to trigger alert rules and exercise the agent's workflow
* Profiles: `constant`, `spike`, `ramp`, `sine`, `ramp-spike`, `agentic`
* Agentic Profile:
  - Normal phase: `--normal-rps` for `--normal-duration`
  - Error phase: `--error-rps` for `--error-duration` (80% hitting `/random-endpoint/error` for 500s)
* Output: Real-time and final stats (RPS, p50/p95/p99, error%)
* Dependencies: Needs Load Balancer up; drives requests that produce necessary metrics and errors

### Phase 7: Optional Testing
**Inject faults via Process Manager**
* Command: `curl -X POST localhost:7070/fault -H "Content-Type: application/json" -d '{"mode":"errors","p_error":0.4}'`
* Role: Dynamically set FAULT modes on running inventory instances to simulate incident scenarios
* Fault Modes: `latency`, `errors`, `cpu`, `oom_safe`, `crash`, `none`
* Implementation: `@app.post("/fault")` calls each instance's `POST /faults` endpoint
* Dependencies: Enhances/replaces traffic-driven incidents; requires running instances


How the alert actually fires
* Rule (alerts.yaml): InventoryHighErrorRate fires when
* sum(rate(http_errors_total{job="inventory"}[1m]))/sum(rate(http_requests_total{job="inventory"}[1m])) > 0.2 for 1 minute.
* Metrics source: Emitted by inventory_service.py in the @app.middleware("http") (REQS and ERRS counters).
* SD and scrape:  prometheus.yaml discovers targets from inventory_targets.json; scrape interval 5s.
* Delivery: Alertmanager forwards to http://127.0.0.1:8088/alerts per alertmanager.yaml

End-to-end runtime flow

* **Traffic** hits the LB (loadbalancer.py), which picks a backend via pick_primary_and_alt() and proxies.
* **Inventory service** (inventory_service.py) handles requests and may:
    * Return 200s for /items/{id} and /put operations.
    * Throw 500s for /random-endpoint or /random-endpoint/error (simulated NPE-like error).
* **Metrics** are recorded (http_requests_total, http_errors_total, durations). Error logs at ERROR level go to logs/inventory_common.log.
* **Prometheus** scrapes metrics from all running inventory instances based on inventory_targets.json.
* **Alert firing**: If error rate > 20% for 1 minute, InventoryHighErrorRate triggers.
* **Alertmanager** sends webhook to on_call_agent.py at POST /alerts.
* **On-call agent**:
    * Instructs LLM to fetch logs and error rate in parallel, and optionally GitHub code.
    * Produces a diagnosis with root cause and next steps (send_diagnosis).
    * Optionally sends a WhatsApp message via Twilio, and later creates a Jira task if the user replies with such a request (/webhook/twilio).
* **Operator loop**: Operator receives the WhatsApp diagnosis and can respond (e.g., “create Jira task and assign to Ravi”), which the agent handles.


Notes and Ports
* **Process Manager**: 7070
* **Inventory Instances**: 8001..8010 (default range)
* **Load Balancer**: 9000
* **Prometheus**: 9090
* **Alertmanager**: 9093
* **On-Call Agent**: 8088

Summary of the Full Flow
* Start **process_manager.py**, scale inventory instances, and thereby populate **inventory_targets.json**.
* Start **loadbalancer.py** so it can pull dynamic backends from the process manager.
* Start Prometheus with **prometheus.yaml** to scrape inventory metrics and load **alerts.yaml**.
* Start Alertmanager with **alertmanager.yaml** to forward alerts to **on_call_agent.py**.
* Start **on_call_agent.py** to process alerts, fetch logs via the LB, compute error rate from Prometheus, and diagnose.
* Drive traffic with **traffic_gen.py** (agentic profile recommended) to generate real 500s.
* When the error rate crosses 20% for 1 minute, an alert fires; the agent responds with an automated diagnosis and optional WhatsApp + Jira integration.
