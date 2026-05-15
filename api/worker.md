# Worker System Documentation (Processing Engine)

---

### Worker System Architecture

* [**Overview**](#overview)

  * [Mental Model](#mental-model)
  * [Core Responsibilities](#core-responsibilities)
  * [Processing Flow](#processing-flow)
* [**Core Concepts**](#core-concepts)

  * [TTL (Time-To-Live)](#ttl-time-to-live)
  * [Idempotency Lock](#idempotency-lock)
  * [Heartbeat](#heartbeat)
  * [Queue Segmentation](#queue-segmentation)
  * [Dead Letter Queue (DLQ)](#dead-letter-queue-dlq)
* [**Architecture Breakdown**](#architecture-breakdown)

  * [Entry Point](#entry-point-srcindexts)
  * [Queue Layer](#queue-layer)
  * [Worker Layer](#worker-layer)
  * [Processing Layer](#processing-layer)
  * [Storage Layer](#storage-layer)
  * [Admin & API Layer](#admin--api-layer)
* [**Processing Pipeline**](#processing-pipeline)

  * [Step-by-Step Execution](#step-by-step-execution)
  * [Video Pipeline](#video-pipeline)
  * [Image Pipeline](#image-pipeline)
* [**Concurrency & Resource Control**](#concurrency--resource-control)

  * [CPU Limiting](#cpu-limiting)
  * [Disk Reservation](#disk-reservation)
  * [User-Level Limits](#user-level-limits)
  * [Upload Throttling](#upload-throttling)
* [**Failure Handling Strategy**](#failure-handling-strategy)

  * [Retry Logic](#retry-logic)
  * [Poison Job Detection](#poison-job-detection)
  * [DLQ Strategy](#dlq-strategy)
* [**Admin API Endpoints**](#admin-api-endpoints)
* [**System Guarantees**](#system-guarantees)
* [**Weak Points & Risks**](#weak-points--risks)
* [**Key Design Decisions**](#key-design-decisions)
* [**Notes**](#notes)

---

# Overview

## Mental Model

This system is a **distributed job processing pipeline**:

```
Client → enqueueFile → Redis → BullMQ → Worker → Process → Upload → API/UI
```

The worker is **not just a processor**, it is:

* a **state machine executor**
* a **resource scheduler**
* a **fault-tolerant pipeline engine**

Reference system flow: 

---

## Core Responsibilities

The worker system is responsible for:

* executing file processing jobs
* maintaining job state in Redis
* ensuring idempotency (no duplicate execution)
* managing CPU, disk, and concurrency limits
* handling retries and failures
* generating previews and final outputs

---

## Processing Flow

```
enqueue → queue → worker picks job
        → lock → download → process → upload
        → update Redis → release resources
```

---

# Core Concepts

## TTL (Time-To-Live)

Used in:

* job metadata
* logs
* locks

Purpose:

* prevents Redis memory leaks
* auto-cleans stale jobs

---

## Idempotency Lock

```
lock:{jobId}
```

Prevents duplicate execution caused by:

* retries
* worker crashes
* race conditions

Mechanism:

```
SET lockKey WORKER_ID PX TTL NX
```

---

## Heartbeat

Runs periodically:

* extends lock TTL
* prevents lock expiry during long jobs

---

## Queue Segmentation

Queues:

* small-files
* medium-files
* large-files
* image-files

Purpose:

* prevents large jobs blocking small ones
* enables priority isolation

---

## Dead Letter Queue (DLQ)

```
dead-letter-queue
```

Stores permanently failed jobs for:

* debugging
* manual retry

---

# Architecture Breakdown

## Entry Point (src/index.ts)

Responsibilities:

* initializes workers
* starts API server
* runs schedulers
* runs background jobs:

  * stuck job recovery
  * disk cleanup

---

## Queue Layer

### enqueueFile()

Responsibilities:

* stores metadata in Redis
* assigns priority
* selects queue
* pushes job to BullMQ

Redis = **source of truth**

---

## Worker Layer

Workers:

| Worker Type    | Queue        | Purpose               |
| -------------- | ------------ | --------------------- |
| fastWorker     | small-files  | high throughput       |
| standardWorker | medium-files | balanced              |
| heavyWorker    | large-files  | controlled processing |
| imageWorker    | image-files  | lightweight tasks     |

---

## Processing Layer

Handled in:

```
worker/handler.ts
```

Responsibilities:

* lock acquisition
* download
* processing (image/video)
* upload
* error handling

---

## Storage Layer

Handles:

* local storage (dev)
* R2 (production)

Includes:

* upload throttling
* distributed upload limiter

---

## Admin & API Layer

Endpoints:

* `/admin`
* `/admin/job/:id`
* `/preview/:id`
* `/admin/dlq`
* `/admin/retry/:id`

Provides:

* system snapshot
* job tracking
* retry controls

---

# Processing Pipeline

## Step-by-Step Execution

1. Acquire lock
2. Allocate resources (CPU, disk, user slot)
3. Download file
4. Normalize input
5. Process file
6. Upload result
7. Update Redis
8. Cleanup
9. Release lock

---

## Video Pipeline

Key steps:

* probe video (ffprobe)
* optional MKV remux
* generate preview clip
* process full video (FFmpeg)
* track progress via `out_time_ms`

Output:

* MP4 (final)
* preview clip (fallback supported)

---

## Image Pipeline

Handled using Sharp:

* resize
* watermark
* format optimization

Output format is dynamically chosen:

* PNG / JPEG / WebP / GIF

---

# Concurrency & Resource Control

## CPU Limiting

Global distributed semaphore:

```
global:cpu
```

Prevents CPU oversubscription across workers.

---

## Disk Reservation

Redis-based reservation:

```
global:disk_reserved
```

Prevents:

* multiple workers consuming disk simultaneously

---

## User-Level Limits

```
user:{userId}:active
```

Ensures:

* fair usage per user tier

---

## Upload Throttling

Global limiter:

```
global:upload_slots
```

Prevents upload bottlenecks.

---

# Failure Handling Strategy

## Retry Logic

Based on:

* file size
* retry count
* error type

---

## Poison Job Detection

If same error repeats:

→ job is moved to DLQ early

---

## DLQ Strategy

Jobs moved when:

* max retries exceeded
* non-recoverable failure

---

# Admin API Endpoints

| Endpoint           | Purpose              |
| ------------------ | -------------------- |
| `/admin`           | full system snapshot |
| `/admin/job/:id`   | job details          |
| `/preview/:id`     | preview URLs         |
| `/admin/dlq`       | failed jobs          |
| `/admin/retry/:id` | retry job            |
| `/job/:id`         | public job status    |

---

# System Guarantees

The system guarantees:

* no duplicate processing (lock)
* bounded resource usage (CPU, disk)
* eventual completion or failure (no silent drops)
* observability via Redis + API

---

# Weak Points & Risks

## Redis Dependency

Assumption:

* Redis is always available

Risk:

* full system failure if Redis goes down

---

## Disk Race Conditions

Even with reservation:

* edge cases possible under high concurrency

---

## Lock TTL Accuracy

If heartbeat fails:

* duplicate execution possible

---

## fs.watch Reliability

Mitigated with fallback scanning, but still imperfect.

---

# Key Design Decisions

## Redis as Source of Truth

Not BullMQ

Reason:

* flexible metadata
* UI-friendly state

---

## HLS Avoidance (Preview Strategy)

Instead of full HLS:

* lightweight preview clips

Reason:

* faster processing
* lower cost

---

## Separate Queues

Ensures fairness and prevents starvation.

---

## Distributed Resource Control

Using Redis:

* CPU
* disk
* uploads

---

# Notes

* Worker is the **core engine of MitFloww**
* System is designed for **horizontal scalability**
* Strong focus on:

  * fault tolerance
  * fairness
  * observability


---

[⬆ Back to Table of Contents](#worker-system-architecture)
