# MitFloww API — Logging, Error Handling & Observability Guide

## 1. Overview & Architecture

The MitFloww API uses a centralized, high-performance structured logging and observability subsystem located at `src/lib/logger/`.

### Core Design Principles
1. **Zero Server Crashes from Recoverable Errors**: All asynchronous operations, background promises, callbacks, and database/storage interactions run inside bounded error boundaries.
2. **End-to-End Correlation**: Every HTTP request is assigned a unique `requestId` and `correlationId` propagated seamlessly through Node.js `AsyncLocalStorage`.
3. **Automated Sensitive-Data Redaction**: Sensitive attributes (passwords, OTPs, session secrets, API keys, tokens, signed S3/R2 signatures, DB credentials) are deeply and recursively stripped before emission.
4. **Environment-Adaptive Output**:
   - **Production (`NODE_ENV=production`)**: Emits single-line structured JSON logs to `stdout` / `stderr`, ready for ingestion by Datadog, AWS CloudWatch, Grafana Loki, or GCP Cloud Logging.
   - **Development (`NODE_ENV=development`)**: Emits clean, color-coded, human-readable console entries with context badges.

```mermaid
flowchart TD
    A[Incoming HTTP Request] --> B[Request Tracing Middleware]
    B -->|Generates X-Request-Id & Correlation-Id| C[AsyncLocalStorage Context]
    C --> D[Request Logger Middleware]
    D --> E[Routes & Controllers]
    E --> F[Services & Repositories]
    F -->|R2 / Postgres / Resend| G[External Providers]
    E -.->|Error Thrown| H[Centralized Error Handler]
    H -->|Describe Safe Error| I[Safe Client Response & Structured Error Log]
    C -.->|Auto-injected Context| J[Centralized Logger]
    J -->|Deep Redaction| K[Sanitizer]
    K -->|Production: JSON / Dev: Colorized| L[Stdout / Stderr]
```

---

## 2. Directory Structure

```text
src/
├── lib/
│   ├── logger/
│   │   ├── index.ts          # Barrel exports
│   │   ├── context.ts        # AsyncLocalStorage request context store
│   │   ├── sanitizer.ts      # Deep redaction for secrets & credential URIs
│   │   └── logger.ts         # Centralized Logger & Scoped Logger factory
│   ├── errors/
│   │   ├── app-error.ts      # Structured operational application error
│   │   └── safe-error-response.ts # Safe error translation & client response
│   └── server/
│       └── process-safety.ts # Uncaught exception & graceful shutdown lifecycle
└── middleware/
    ├── request-tracing.ts    # Request & Correlation ID injection
    ├── request-logger.ts     # HTTP request lifecycle logging
    └── error-handler.ts      # Express error boundary with headersSent guard
```

---

## 3. Request Context & Correlation

### How It Works
When a request enters the API, `requestTracingMiddleware` generates or adopts:
- `X-Request-Id`: Unique per HTTP request (e.g. `req_a1b2c3d4e5f6`).
- `X-Correlation-Id`: Preserves distributed tracing across upstream clients or workers.
- Context is stored in `AsyncLocalStorage` via `requestContextStorage.run(...)`.

### Automatic Context Injection
Any call to `logger.info`, `logger.warn`, `logger.error` automatically acquires:
- `requestId`
- `correlationId`
- `userId` (when authenticated)
- `method`
- `path`

There is **no need to manually pass `req` or `requestId`** to deep service functions!

```ts
import { logger } from "@/lib/logger";

// Automatically contains requestId, userId, path, and method in the log output:
logger.info("Deliverable version created", { fileId: "file_123", version: 2 });
```

---

## 4. Scoped Loggers for Subsystems

For distinct domain services or background workers, use `createScopedLogger(componentName)`:

```ts
import { createScopedLogger } from "@/lib/logger";

const scopedLogger = createScopedLogger("billing-service");

// Emits logs with { component: "billing-service" }
scopedLogger.info("Subscription invoice generated", { invoiceId: "inv_456" });
scopedLogger.warn("Payment gateway retry initiated", { attempt: 2, gateway: "razorpay" });
scopedLogger.error("Webhook signature mismatch", { err: signatureError });
```

---

## 5. Automated Data Redaction & Sanitization

The `sanitizer.ts` module recursively redacts:

1. **30+ Sensitive Object Keys**:
   `password`, `passwordHash`, `otp`, `token`, `accessToken`, `refreshToken`, `sessionSecret`, `apiKey`, `secretAccessKey`, `privateKey`, `authorization`, `cookie`, `cardNumber`, etc.
2. **URI Embedded Credentials**:
   `postgres://user:SuperSecretPassword@host:5432/db` $\rightarrow$ `postgres://***:***@host:5432/db`
3. **Presigned Cloudflare R2 / AWS S3 Query Signatures**:
   `?X-Amz-Signature=abcd1234efgh` $\rightarrow$ `?X-Amz-Signature=***`
4. **Circular Object Graphs**:
   Safely detected and marked as `[CircularReference]` without stack overflow.

---

## 6. Error Handling & Failure Isolation

### AppError Model
Use `AppError` for expected business rule violations and operational failures:

```ts
import { AppError } from "@/lib/errors/app-error";

throw new AppError("File not found or access denied.", 404, "file_not_found");
```

### Safe Client Responses
Internal database errors, raw SQL queries, stack traces, and filesystem paths are **never sent to the client**.

`safe-error-response.ts` converts errors into sanitized client responses:

```json
{
  "error": {
    "code": "file_not_found",
    "message": "File not found or access denied.",
    "messageKey": "common.errors.notFound",
    "requestId": "req_8f439ed48f644f2c91f0"
  }
}
```

---

## 7. Process Protection & Graceful Shutdown

Located in `src/lib/server/process-safety.ts`, the process safety subsystem handles:

1. **`uncaughtException`**:
   - Emits a `FATAL` structured log with full error stack.
   - Triggers graceful shutdown to allow in-flight requests to complete before process exit.
2. **`unhandledRejection`**:
   - Emits a structured `ERROR` log with context.
   - Prevents unhandled promise rejections from killing the Node.js API process.
3. **Graceful Shutdown (`SIGTERM` / `SIGINT`)**:
   - Stops HTTP server from accepting new connections.
   - Drains PostgreSQL connection pools (`pool.end()`).
   - Executes registered cleanup hooks.
   - Enforces a 10-second hard shutdown timeout.

---

## 8. Log Levels

Configured via the `LOG_LEVEL` environment variable (`trace` | `debug` | `info` | `warn` | `error` | `fatal`).

| Level | Severity | Usage |
| :--- | :--- | :--- |
| **`trace`** | 10 | Extremely verbose debugging & byte-level network tracing. |
| **`debug`** | 20 | Detailed diagnostic information (health checks, dev queries). |
| **`info`** | 30 | High-level business milestones (user login, file upload, payments). |
| **`warn`** | 40 | Non-fatal anomalies, retries, 4xx client errors, cache fallbacks. |
| **`error`** | 50 | 5xx server errors, provider failures, failed background jobs. |
| **`fatal`** | 60 | Unrecoverable process failures requiring shutdown. |

---

## 9. Verification & Testing

Run the test suite to verify logging, redaction, and error isolation:

```bash
# Run unit & middleware tests
pnpm test

# Run TypeScript typecheck
pnpm typecheck

# Build production bundle
pnpm build
```
