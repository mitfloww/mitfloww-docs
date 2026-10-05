# MitFloww Internal Operations & Observability Console Architecture

## 1. Overview
The MitFloww architecture establishes **Scheduler** as the single authoritative home for all internal/admin operations and multi-service operational observability.

```
MitFloww
├── Web (Port 3000)
│   └── Customer-facing Next.js application (Zero Admin surface)
├── API (Port 4001)
│   └── Customer/business Express API (Zero Admin routes)
├── Worker (Port 4000)
│   └── Background media processing service (Stateless BullMQ worker)
└── Scheduler (Port 4002)
    └── Internal Operations / Admin Console
        ├── Overview & Core Infrastructure Dashboard
        ├── Scheduler Jobs & Cooperative Controls
        ├── Run History & Failure Logs
        ├── Web & System Services Health
        ├── Worker Admin (Queues, Active Jobs, DLQ, Previews, Retries)
        ├── Operational Logs (Service-Separated: Web, API, Worker, Scheduler)
        └── Admin Audit Trail
```

## 2. Admin Migration & Cleanup
- **Web Application**: The obsolete `/admin` route (`src/app/admin/page.tsx`) and `src/features/admin/` were completely removed. Recharts dependency was eliminated. Admin translations were cleanly excised without affecting customer-facing keys.
- **API Service**: The obsolete `/admin` router (`src/routes/admin.ts`) was unmounted and deleted.
- **Scheduler**: Hosted at `http://localhost:4002/` with privileged access controlled via `SCHEDULER_ADMIN_KEY` and `scheduler_admin_token` HttpOnly cookie / Bearer authorization.

## 3. Worker Admin Data-Fetching Optimization
- **Previous Mechanism**: Aggressive 1-second continuous WebSocket broadcasts and repetitive HTTP polling from the client, causing server overhead and port collisions (4002).
- **Optimized Mechanism**:
  1. **Server-Side Caching**: Scheduler caches Worker status responses for 4,000ms.
  2. **In-Flight Deduplication**: Multiple simultaneous UI requests share a single underlying Worker HTTP request.
  3. **Adaptive Polling**: Polling operates on an 8-second interval and automatically suspends when the tab is blurred or hidden (`document.visibilitychange`), resuming on focus.
  4. **Direct Communication**: Scheduler talks directly to Worker HTTP API on port 4000 without proxying through `api`.
  5. **Estimated Traffic Reduction**: >85% reduction in API/Worker network traffic and zero port collisions.

## 4. Multi-Service Operational Logging Architecture
Each service writes both to local/stream logs (stdout/stderr / file) and asynchronously persists structured records into dedicated PostgreSQL tables in the `mitfloww` schema:

| Service | Persistent DB Table | Local Stream | Key Identifiers |
|---|---|---|---|
| **API** | `mitfloww.api_logs` | stdout / pino format | `request_id`, `correlation_id`, `user_id`, `operation` |
| **Web** | `mitfloww.web_logs` | stdout / json format | `request_id`, `correlation_id`, `session_id`, `path` |
| **Worker** | `mitfloww.worker_logs` | `logs/worker-YYYY-MM-DD.log` | `job_id`, `job_name`, `file_id`, `stage` |
| **Scheduler** | `mitfloww.scheduler_logs` | stdout / structured json | `execution_id`, `job_name`, `trigger` |

### 4.1 Reliability & Failure Isolation
- **Total Decoupling**: Database log insertion is asynchronous and fire-and-forget. A failure in PostgreSQL logging never throws or interrupts business operations, request handling, or media processing.
- **Centralized Redaction**: Passwords, tokens, cookies, authorization headers, and secrets are stripped prior to persistence.
- **Log Retention Job**: `LogRetentionJob` runs automatically in the Scheduler every 12 hours, enforcing configurable retention days per service (`*_LOG_RETENTION_DAYS`, default 30 days) and respecting `DRY_RUN` mode.
