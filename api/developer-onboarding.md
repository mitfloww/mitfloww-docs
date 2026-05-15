# Developer Onboarding Guide

This document is a practical onboarding guide for a developer joining this project.

It is written for someone who is comfortable with .NET, knows Angular a little, and is new to Next.js App Router, React Server Components, this project's feature-based structure, and the file upload / deliverable flow.

Based on the code in this repository, the app appears to provide:

- a dashboard-style shell with pages for dashboard, projects, analytics, team, admin, and health
- project and deliverable screens
- browser-based file upload flows
- revision and payment/preview settings stored with file metadata
- R2-backed binary storage
- PostgreSQL-backed file metadata through Drizzle
- several API route families for health checks, files, upload sessions, multipart uploads, metrics, and admin worker proxying

<a id="quick-navigation-and-glossary"></a>
## Quick Navigation and Glossary

### Major Sections

| Section | What it covers | Key files |
| --- | --- | --- |
| [Mental Model](#mental-model) | Next.js App Router explained in .NET/Angular terms | [src/app/layout.tsx](../src/app/layout.tsx), [src/app/api/files/route.ts](../src/app/api/files/route.ts) |
| [High-Level Architecture](#high-level-architecture) | The main folders and how requests/data move through the app | [src/app/(dashboard)/layout.tsx](<../src/app/(dashboard)/layout.tsx>), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) |
| [Page and Route Inventory](#page-and-route-inventory) | What each page route does and where it starts | [src/app/(dashboard)/page.tsx](<../src/app/(dashboard)/page.tsx>), [src/app/admin/page.tsx](../src/app/admin/page.tsx) |
| [API Inventory](#api-inventory) | Every API route under `src/app/api` | [src/app/api/files/route.ts](../src/app/api/files/route.ts), [src/app/api/file-uploads/commit/route.ts](../src/app/api/file-uploads/commit/route.ts) |
| [File Upload Architecture](#file-upload-architecture) | The most important section for uploads, multipart, commit, cleanup, and deliverables | [src/features/files/lib/upload-client.ts](../src/features/files/lib/upload-client.ts), [src/features/files/components/upload-file-modal.tsx](../src/features/files/components/upload-file-modal.tsx), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) |
| [Projects and Deliverables Flow](#projects-and-deliverables-flow) | How projects, deliverables, revision settings, and project details fit together | [src/features/projects/components/project-details.client.tsx](../src/features/projects/components/project-details.client.tsx), [src/features/files/components/deliverable-card.tsx](../src/features/files/components/deliverable-card.tsx) |
| [Admin Flow](#admin-flow) | Admin page and worker proxy routes | [src/features/admin/components/admin-dashboard.tsx](../src/features/admin/components/admin-dashboard.tsx), [src/app/api/admin/route.ts](../src/app/api/admin/route.ts) |
| [Health Checks](#health-checks) | Local/dev verification endpoints and UI | [src/features/health/components/health-check-page.tsx](../src/features/health/components/health-check-page.tsx), [src/app/api/health/db/route.ts](../src/app/api/health/db/route.ts) |
| [Database Layer](#database-layer) | Drizzle client, schema, repository pattern | [src/lib/db/client.ts](../src/lib/db/client.ts), [src/lib/repositories/file-repository.ts](../src/lib/repositories/file-repository.ts) |
| [API Response and Error Handling](#api-response-and-error-handling) | Standard API wrapper and `AppError` usage | [src/lib/api/route.ts](../src/lib/api/route.ts), [src/lib/errors/app-error.ts](../src/lib/errors/app-error.ts) |
| [Validation](#validation) | Where validation lives and how to extend it safely | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts) |
| [i18n and Localization](#i18n-and-localization) | Locale detection, message files, and translation hooks | [src/i18n/provider.tsx](../src/i18n/provider.tsx), [src/i18n/messages/en.ts](../src/i18n/messages/en.ts) |
| [UI and Component System](#ui-and-component-system) | Shared UI components, dialogs, toasts, skeletons | [src/components/ui/button.tsx](../src/components/ui/button.tsx), [src/components/ui/confirm-action-dialog.tsx](../src/components/ui/confirm-action-dialog.tsx) |
| [State Management and Providers](#state-management-and-providers) | What each provider owns and where it is mounted | [src/features/projects/providers/projects-provider.tsx](../src/features/projects/providers/projects-provider.tsx), [src/features/files/providers/files-provider.tsx](../src/features/files/providers/files-provider.tsx) |
| [Query Helpers](#query-helpers) | Centralized search/sort/pagination logic | [src/lib/query/files.ts](../src/lib/query/files.ts), [src/lib/query/pagination.ts](../src/lib/query/pagination.ts) |
| [Styling](#styling) | Tailwind v4 tokens, class merging, and consistency rules | [src/app/globals.css](../src/app/globals.css), [src/lib/utils/cn.ts](../src/lib/utils/cn.ts) |
| [Middleware](#middleware) | Current middleware status and where it would go later |  |
| [Development Workflow](#development-workflow) | How to run, lint, build, and verify the app | [package.json](../package.json), [.env.example](../.env.example) |
| [Environment Variables](#environment-variables) | What `process.env` values are used today | [src/lib/db/client.ts](../src/lib/db/client.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| [How to Add a New Feature](#how-to-add-a-new-feature) | Step-by-step patterns for adding pages, APIs, DB logic, UI, validation, and i18n |  |
| [How to Modify Upload Behavior](#how-to-modify-existing-file-upload-behavior) | The safest places to change upload behavior without breaking multipart or commit | [src/features/files/lib/upload-client.ts](../src/features/files/lib/upload-client.ts), [src/lib/validation/files.ts](../src/lib/validation/files.ts) |
| [Architecture Rules](#architecture-rules) | The rules worth following before you make bigger changes |  |
| [Common Pitfalls](#common-pitfalls-for-net--angular-developers) | Things that often trip up .NET / Angular developers in App Router |  |
| [Detailed Glossary](#detailed-glossary) | Longer definitions of project and Next.js terms |  |
| [Onboarding Checklist](#final-onboarding-checklist) | A practical checklist for your first day or two |  |

### Compact Glossary

| Term | Short meaning | See |
| --- | --- | --- |
| App Router | Next.js file-system routing under `src/app` | [Mental Model](#mental-model) |
| Server Component | Default App Router component that runs on the server | [Mental Model](#mental-model) |
| Client Component | React component marked with `"use client"` | [Mental Model](#mental-model) |
| Route Handler | `route.ts` server endpoint under `app/api` | [Mental Model](#mental-model) |
| Layout | Shared shell for nested routes | [High-Level Architecture](#high-level-architecture) |
| Loading UI | `loading.tsx` shown automatically during route loading | [Page and Route Inventory](#page-and-route-inventory) |
| Dynamic Route | Folder like `[id]` that becomes a route parameter | [Mental Model](#mental-model) |
| Route Group | Folder like `(dashboard)` that does not appear in the URL | [Mental Model](#mental-model) |
| Provider | React context owner for shared state | [State Management and Providers](#state-management-and-providers) |
| DTO | Typed object passed across API/service layers | [API Response and Error Handling](#api-response-and-error-handling) |
| Repository | DB access layer | [Database Layer](#database-layer) |
| Service | Business logic layer | [Database Layer](#database-layer) |
| R2 | Cloudflare object storage used for uploaded binaries | [File Upload Architecture](#file-upload-architecture) |
| Multipart Upload | Large-file upload split into parts | [File Upload Architecture](#file-upload-architecture) |
| Orphan Upload | Storage object/upload with no matching DB record | [File Upload Architecture](#file-upload-architecture) |
| DLQ | Dead-letter queue concept exposed by admin worker APIs | [Admin Flow](#admin-flow) |
| Toast | Temporary notification UI | [UI and Component System](#ui-and-component-system) |
| Skeleton | Placeholder loading component | [UI and Component System](#ui-and-component-system) |
| Deliverable | Current UI view of a committed file record | [Projects and Deliverables Flow](#projects-and-deliverables-flow) |
| Revision | Revision limit and extra revision cost metadata saved on files | [File Upload Architecture](#file-upload-architecture) |
| Upload Session | Client-side upload handshake result containing bucket/key/uploadId | [File Upload Architecture](#file-upload-architecture) |

<a id="mental-model"></a>
## Mental Model for a .NET / Angular Developer

If you think in ASP.NET MVC + Web API + Angular terms, the easiest mapping is:

- `src/app/**/page.tsx` is closest to a route page or Angular route entry component.
- `src/app/**/layout.tsx` is the shared shell for a route branch, similar to a layout page or a top-level Angular shell component.
- `src/app/api/**/route.ts` is a backend endpoint, closer to a minimal API endpoint than a React component.
- `loading.tsx` is automatic route-level loading UI for that route segment.
- `[id]` is a route parameter.
- `(dashboard)` is a route group. It organizes files, but it does not show up in the URL.

Examples from this repo:

- `/` comes from [src/app/(dashboard)/page.tsx](<../src/app/(dashboard)/page.tsx>), not from `src/app/page.tsx`.
- `/projects/[id]` comes from [src/app/(dashboard)/projects/[id]/page.tsx](<../src/app/(dashboard)/projects/[id]/page.tsx>).
- `/api/files` comes from [src/app/api/files/route.ts](../src/app/api/files/route.ts).

### Folder Meaning in This Repo

- `app` is routing, layouts, and API entrypoints.
- `features` is domain code. Think "project feature", "files feature", "admin feature".
- `components` is reusable shared UI and layout pieces.
- `lib` is server/business/infrastructure code: DB, validation, DTOs, repositories, services, storage, query helpers.

### Server Component vs Client Component

By default, App Router components are Server Components.

That means:

- they can run async server code
- they can import server-safe modules
- they cannot use browser-only hooks like `useState`, `useEffect`, `useRouter`, or `window`

If a file starts with `"use client"`, it becomes a Client Component.

That means:

- it can use React state/effects and browser APIs
- it runs in the browser
- it must not import server-only modules like `src/lib/db/*` or `src/lib/storage/*`

Examples in this repo:

- [src/features/projects/components/projects-page.client.tsx](../src/features/projects/components/projects-page.client.tsx) is a Client Component because it uses state, tabs, and modal interactions.
- [src/features/files/components/upload-file-modal.tsx](../src/features/files/components/upload-file-modal.tsx) is a Client Component because it handles file input, drag/drop, upload progress, and local cleanup logic.
- [src/app/(dashboard)/projects/[id]/page.tsx](<../src/app/(dashboard)/projects/[id]/page.tsx>) is a Server Component page that reads `params` and passes `projectId` into a client feature component.

### When to Put Code on the Server vs the Client

Use server-side code when:

- you need DB access
- you need storage access
- you need secrets or `process.env`
- you are defining an API route

Use client-side code when:

- you need `useState`, `useEffect`, or browser events
- you need file input or drag/drop
- you need local progress bars, modals, toasts, or optimistic UI

In this project, the common pattern is:

- `page.tsx` stays thin
- feature UI lives in `features/<domain>/components`
- browser actions call `/api/...`
- route handlers validate input and delegate to `lib/services/*`

<a id="high-level-architecture"></a>
## High-Level Architecture

### Main Layers

- `src/app`
  - routing
  - layouts
  - API route entrypoints
- `src/features`
  - domain-specific UI, hooks, and providers
  - examples: `projects`, `files`, `admin`
- `src/components`
  - shared UI and layout building blocks
  - examples: buttons, cards, dialogs, skeletons, navbar
- `src/lib`
  - backend/domain infrastructure
  - DB client and schema
  - repositories
  - services
  - DTOs
  - validation
  - storage
  - query helpers
- `src/config`
  - upload limits and input limits
- `src/i18n`
  - localization config, messages, provider, locale switching
- `src/hooks`
  - shared React hooks like toast/currency
- `src/types`
  - shared TypeScript types used in UI and DTO mapping

### Important Layout Composition

- Root layout: [src/app/layout.tsx](../src/app/layout.tsx)
  - global CSS
  - locale resolution
  - localization provider
  - toast provider
- Dashboard layout: [src/app/(dashboard)/layout.tsx](<../src/app/(dashboard)/layout.tsx>)
  - navbar
  - `ProjectsProvider`
  - `FilesProvider`

### Request/Data Flow

```mermaid
flowchart LR
    A[Browser] --> B[Page or Client Component]
    B --> C[Feature Component / Provider]
    C --> D[API Route Handler]
    D --> E[Validation]
    E --> F[Service]
    F --> G[Repository]
    G --> H[(PostgreSQL)]
    F --> I[(R2 Storage)]
    H --> F
    I --> F
    F --> D
    D --> C
    C --> A
```

### One Important Upload-Specific Note

There are two file API families in the repo:

- `/api/files/...`
  - DB-first file flow
  - create metadata row first, then upload content by file id
- `/api/file-uploads/...`
  - upload-session/object/commit flow
  - upload to managed storage first, then commit file rows later

Current implementation appears to use the second family in the actual upload modal and file provider.

<a id="page-and-route-inventory"></a>
## Page and Route Inventory

| Route | Page file | Likely responsibility | Main feature component | Likely render style | Loading / skeleton support |
| --- | --- | --- | --- | --- | --- |
| `/` | [src/app/(dashboard)/page.tsx](<../src/app/(dashboard)/page.tsx>) | Dashboard landing page | [src/features/dashboard/components/dashboard-page-content.tsx](../src/features/dashboard/components/dashboard-page-content.tsx) | Server page composing a Client Component | None found |
| `/analytics` | [src/app/(dashboard)/analytics/page.tsx](<../src/app/(dashboard)/analytics/page.tsx>) | Analytics page placeholder | [src/features/analytics/components/analytics-page-content.tsx](../src/features/analytics/components/analytics-page-content.tsx) | Server page composing a Client Component | None found |
| `/projects` | [src/app/(dashboard)/projects/page.tsx](<../src/app/(dashboard)/projects/page.tsx>) | Project list, project creation, filtering | [src/features/projects/components/projects-page.client.tsx](../src/features/projects/components/projects-page.client.tsx) | Server page composing a Client Component | [src/app/(dashboard)/projects/loading.tsx](<../src/app/(dashboard)/projects/loading.tsx>), [src/components/skeletons/page-header-skeleton.tsx](../src/components/skeletons/page-header-skeleton.tsx), [src/components/skeletons/grid-skeleton.tsx](../src/components/skeletons/grid-skeleton.tsx) |
| `/projects/[id]` | [src/app/(dashboard)/projects/[id]/page.tsx](<../src/app/(dashboard)/projects/[id]/page.tsx>) | Project details, deliverables, upload modal | [src/features/projects/components/project-details.client.tsx](../src/features/projects/components/project-details.client.tsx) | Async server page passing `projectId` into a Client Component | [src/app/(dashboard)/projects/[id]/loading.tsx](<../src/app/(dashboard)/projects/[id]/loading.tsx>), [src/features/projects/skeletons/project-details-header-skeleton.tsx](../src/features/projects/skeletons/project-details-header-skeleton.tsx), [src/features/projects/skeletons/project-details-card-skeleton.tsx](../src/features/projects/skeletons/project-details-card-skeleton.tsx), [src/components/skeletons/deliverable-card-skeleton.tsx](../src/components/skeletons/deliverable-card-skeleton.tsx) |
| `/team` | [src/app/(dashboard)/team/page.tsx](<../src/app/(dashboard)/team/page.tsx>) | Team page placeholder | [src/features/team/components/team-page-content.tsx](../src/features/team/components/team-page-content.tsx) | Server page composing a Client Component | None found |
| `/admin` | [src/app/admin/page.tsx](../src/app/admin/page.tsx) | Realtime admin monitoring and worker preview UI | [src/features/admin/components/admin-dashboard.tsx](../src/features/admin/components/admin-dashboard.tsx) | Server page composing a Client Component | Component has its own loading state; no route `loading.tsx` found |
| `/health` | [src/app/health/page.tsx](../src/app/health/page.tsx) | Manual health-check UI for DB ping and CRUD test | [src/features/health/components/health-check-page.tsx](../src/features/health/components/health-check-page.tsx) | Server page composing a Client Component | Component has its own loading state; no route `loading.tsx` found |

### Notes About Current Pages

- `dashboard`, `analytics`, and `team` are currently simple placeholder-style pages.
- `projects` is the main user flow today.
- `admin` and `health` live outside the `(dashboard)` route group, so they do not use the dashboard layout/navbar.
- Current implementation appears to keep the project list in client memory inside `ProjectsProvider`. Deep-linking to an unknown `/projects/[id]` currently shows the loading UI rather than a clear not-found state. TODO: confirm intended behavior.

<a id="api-inventory"></a>
## API Inventory

### Health and Metrics Routes

| HTTP path | File path | Likely purpose | Expected input source | Expected output shape | Related files |
| --- | --- | --- | --- | --- | --- |
| `/api/health` | [src/app/api/health/route.ts](../src/app/api/health/route.ts) | Simple DB connectivity check using `SELECT 1` | None | Raw JSON like `{ status, db, schema, timestamp }` or `{ status, error }` | [src/lib/db/client.ts](../src/lib/db/client.ts) |
| `/api/health/db` | [src/app/api/health/db/route.ts](../src/app/api/health/db/route.ts) | Create/read/delete test against `health_checks` table | None | Raw JSON like `{ status, steps, schema }` or `{ status, error }` | [src/lib/db/client.ts](../src/lib/db/client.ts), [src/lib/db/schema/health.ts](../src/lib/db/schema/health.ts) |
| `/api/projects/metrics` | [src/app/api/projects/metrics/route.ts](../src/app/api/projects/metrics/route.ts) | Counts deliverables per project and paid deliverables per project | Query string `projectIds=...` | `{ data: Array<{ projectId, totalDeliverablesCount, paidDeliverablesCount }> }` | [src/lib/api/route.ts](../src/lib/api/route.ts), [src/lib/db/schema/files.ts](../src/lib/db/schema/files.ts) |

### Record-First File Routes

| HTTP path | File path | Likely purpose | Expected input source | Expected output shape | Related files |
| --- | --- | --- | --- | --- | --- |
| `/api/files` `GET` | [src/app/api/files/route.ts](../src/app/api/files/route.ts) | List file records with filters, paging, sorting | Query string | `{ data: FileDTO[], meta: { count, filters, pagination, sorting } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/repositories/file-repository.ts](../src/lib/repositories/file-repository.ts) |
| `/api/files` `POST` | [src/app/api/files/route.ts](../src/app/api/files/route.ts) | Create a pending file record before content upload | JSON body | `{ data: FileDTO }` with HTTP 201 | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) |
| `/api/files/[id]` `GET` | [src/app/api/files/[id]/route.ts](<../src/app/api/files/[id]/route.ts>) | Get one file record by id | Route param | `{ data: FileDTO }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) |
| `/api/files/[id]` `PATCH` | [src/app/api/files/[id]/route.ts](<../src/app/api/files/[id]/route.ts>) | Update file metadata | Route param + JSON body | `{ data: FileDTO }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/repositories/file-repository.ts](../src/lib/repositories/file-repository.ts) |
| `/api/files/[id]` `DELETE` | [src/app/api/files/[id]/route.ts](<../src/app/api/files/[id]/route.ts>) | Soft-delete DB record and delete stored object | Route param | `{ data: DeletedFileDTO }` | [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts), [src/lib/repositories/file-repository.ts](../src/lib/repositories/file-repository.ts) |
| `/api/files/[id]/content` `PUT` | [src/app/api/files/[id]/content/route.ts](<../src/app/api/files/[id]/content/route.ts>) | Upload file body for a pending DB file record | Route param + upload stream + headers | `{ data: FileDTO }` | [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/files/[id]/content` `GET` | [src/app/api/files/[id]/content/route.ts](<../src/app/api/files/[id]/content/route.ts>) | Stream stored file content back to the browser | Route param | Raw streamed response with `Content-Type`, optional `Content-Length`, optional `ETag` | [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/files/[id]/cancel` `POST` | [src/app/api/files/[id]/cancel/route.ts](<../src/app/api/files/[id]/cancel/route.ts>) | Cancel a file upload and optionally abort multipart | Route param + JSON body | `{ data: FileDTO }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) |
| `/api/files/[id]/multipart` `POST` | [src/app/api/files/[id]/multipart/route.ts](<../src/app/api/files/[id]/multipart/route.ts>) | Initiate multipart upload for an existing file record | Route param | `{ data: { bucket, fileId, multipartThresholdBytes, partSizeBytes, storageKey, totalParts, uploadId } }` with HTTP 201 | [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/uploads/multipart.ts](../src/lib/uploads/multipart.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/files/[id]/multipart/parts/[partNumber]` `PUT` | [src/app/api/files/[id]/multipart/parts/[partNumber]/route.ts](<../src/app/api/files/[id]/multipart/parts/[partNumber]/route.ts>) | Upload one multipart part for an existing file record | Route param + query `uploadId` + upload stream | `{ data: { etag, partNumber } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/uploads/multipart.ts](../src/lib/uploads/multipart.ts) |
| `/api/files/[id]/multipart/complete` `POST` | [src/app/api/files/[id]/multipart/complete/route.ts](<../src/app/api/files/[id]/multipart/complete/route.ts>) | Complete multipart upload for an existing file record | Route param + JSON body | `{ data: FileDTO }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) |
| `/api/files/[id]/multipart/abort` `POST` | [src/app/api/files/[id]/multipart/abort/route.ts](<../src/app/api/files/[id]/multipart/abort/route.ts>) | Abort multipart upload for an existing file record | Route param + JSON body | `{ data: { aborted, bucket, fileId, storageKey, uploadId, uploadStatus } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) |

### Upload-Session / Object / Commit Routes

| HTTP path | File path | Likely purpose | Expected input source | Expected output shape | Related files |
| --- | --- | --- | --- | --- | --- |
| `/api/file-uploads/sessions` `POST` | [src/app/api/file-uploads/sessions/route.ts](../src/app/api/file-uploads/sessions/route.ts) | Start an upload session and generate managed storage key info | JSON body | `{ data: UploadSessionDTO }` with HTTP 201 | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/uploads/final-storage-keys.ts](../src/lib/uploads/final-storage-keys.ts) |
| `/api/file-uploads/object` `DELETE` | [src/app/api/file-uploads/object/route.ts](../src/app/api/file-uploads/object/route.ts) | Delete an uploaded managed object and optionally abort multipart first | JSON body | `{ data: { bucket, deleted, storageKey, uploadId } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/file-uploads/content` `PUT` | [src/app/api/file-uploads/content/route.ts](../src/app/api/file-uploads/content/route.ts) | Upload a small file directly to managed storage without a DB record yet | Query string + upload stream | `{ data: { bucket, storageKey, uploadId: null } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/file-uploads/content` `GET` | [src/app/api/file-uploads/content/route.ts](../src/app/api/file-uploads/content/route.ts) | Return a presigned or direct upload URL for object upload | Query string | `{ data: { url, expiresAt } }` | [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/file-uploads/commit` `POST` | [src/app/api/file-uploads/commit/route.ts](../src/app/api/file-uploads/commit/route.ts) | Commit uploaded storage objects into DB file records | JSON body | `{ data: { committed: [...], failed: [...] } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/repositories/file-repository.ts](../src/lib/repositories/file-repository.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/file-uploads/finalize` | Folder exists at `src/app/api/file-uploads/finalize`, but no `route.ts` is present | Structure placeholder only | None | Not implemented |  |
| `/api/file-uploads/multipart/parts` `GET` | [src/app/api/file-uploads/multipart/parts/route.ts](../src/app/api/file-uploads/multipart/parts/route.ts) | Intentionally returns not found for the collection route | None | Wrapped not-found error response | [src/lib/api/route.ts](../src/lib/api/route.ts), [src/lib/errors/app-error.ts](../src/lib/errors/app-error.ts) |
| `/api/file-uploads/multipart/parts/[partNumber]` `GET` | [src/app/api/file-uploads/multipart/parts/[partNumber]/route.ts](<../src/app/api/file-uploads/multipart/parts/[partNumber]/route.ts>) | Return presigned URL for one multipart part | Route param + query string | `{ data: { partNumber, url, expiresAt } }` | [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/file-uploads/multipart/parts/[partNumber]` `PUT` | [src/app/api/file-uploads/multipart/parts/[partNumber]/route.ts](<../src/app/api/file-uploads/multipart/parts/[partNumber]/route.ts>) | Upload one multipart part through the app route | Route param + query string + upload stream | `{ data: { etag, partNumber } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/uploads/multipart.ts](../src/lib/uploads/multipart.ts) |
| `/api/file-uploads/multipart/complete` `POST` | [src/app/api/file-uploads/multipart/complete/route.ts](../src/app/api/file-uploads/multipart/complete/route.ts) | Complete multipart upload for an upload session | JSON body | `{ data: { bucket, storageKey, uploadId } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/file-uploads/multipart/abort` `POST` | [src/app/api/file-uploads/multipart/abort/route.ts](../src/app/api/file-uploads/multipart/abort/route.ts) | Abort multipart upload for an upload session | JSON body | `{ data: { aborted, bucket, reason, storageKey, uploadId } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) |
| `/api/file-uploads/temp/content` | Folder exists at `src/app/api/file-uploads/temp/content`, but no `route.ts` is present | Structure placeholder only | None | Not implemented |  |
| `/api/file-uploads/temp/multipart/parts/[partNumber]` | Folder exists at `src/app/api/file-uploads/temp/multipart/parts/[partNumber]`, but no `route.ts` is present | Structure placeholder only | None | Not implemented |  |
| `/api/file-uploads/temp/multipart/complete` | Folder exists at `src/app/api/file-uploads/temp/multipart/complete`, but no `route.ts` is present | Structure placeholder only | None | Not implemented |  |
| `/api/file-uploads/temp/multipart/abort` | Folder exists at `src/app/api/file-uploads/temp/multipart/abort`, but no `route.ts` is present | Structure placeholder only | None | Not implemented |  |

### Admin Routes

| HTTP path | File path | Likely purpose | Expected input source | Expected output shape | Related files |
| --- | --- | --- | --- | --- | --- |
| `/api/admin` | [src/app/api/admin/route.ts](../src/app/api/admin/route.ts) | Proxy to external worker admin endpoint | None | Raw proxied JSON from `http://localhost:4000/admin` | [src/features/admin/components/admin-dashboard.tsx](../src/features/admin/components/admin-dashboard.tsx) |
| `/api/admin/dlq` | [src/app/api/admin/dlq/route.ts](../src/app/api/admin/dlq/route.ts) | Proxy to worker DLQ endpoint | None | Raw proxied JSON from `http://localhost:4000/admin/dlq` | [src/features/admin/components/admin-dashboard.tsx](../src/features/admin/components/admin-dashboard.tsx) |
| `/api/admin/job/[id]` | [src/app/api/admin/job/[id]/route.ts](<../src/app/api/admin/job/[id]/route.ts>) | Proxy to worker job-detail endpoint | Route param | Raw proxied JSON or `{ error }` with HTTP 500 | [src/features/admin/components/admin-dashboard.tsx](../src/features/admin/components/admin-dashboard.tsx) |
| `/api/admin/preview/[id]` | [src/app/api/admin/preview/[id]/route.ts](<../src/app/api/admin/preview/[id]/route.ts>) | Proxy to worker preview endpoint | Route param | Raw proxied JSON or `{ error }` with HTTP 500 | [src/features/admin/components/admin-dashboard.tsx](../src/features/admin/components/admin-dashboard.tsx) |
| `/api/admin/retry/[id]` | [src/app/api/admin/retry/[id]/route.ts](<../src/app/api/admin/retry/[id]/route.ts>) | Proxy to worker retry endpoint | Route param | Raw proxied JSON or `{ error }` with HTTP 500 | [src/features/admin/components/admin-dashboard.tsx](../src/features/admin/components/admin-dashboard.tsx) |
| `/api/admin/file-uploads/orphans` `GET` / `POST` | [src/app/api/admin/file-uploads/orphans/route.ts](../src/app/api/admin/file-uploads/orphans/route.ts) | Scan or clean orphaned R2 objects/multipart uploads without matching DB records | Query string or JSON body | `{ data: { dryRun, olderThanHours, checkedObjects, deletedObjects, checkedMultipartUploads, abortedMultipartUploads, items } }` | [src/lib/validation/files.ts](../src/lib/validation/files.ts), [src/lib/services/file-service.ts](../src/lib/services/file-service.ts), [src/lib/storage/r2.ts](../src/lib/storage/r2.ts), [src/lib/repositories/file-repository.ts](../src/lib/repositories/file-repository.ts) |

### API Consistency Notes

- Most file/upload/project routes use the wrapper from [src/lib/api/route.ts](../src/lib/api/route.ts).
- Health routes do not use that wrapper. They return raw JSON.
- Admin proxy routes also do not use that wrapper. They return raw worker JSON or ad hoc `{ error }` responses.
- `GET /api/admin/retry/[id]` is a state-changing operation exposed as `GET`. Treat it carefully.

<a id="file-upload-architecture"></a>
## File Upload Architecture

This is the most important section if you will touch uploads.

### The Big Picture

The active browser upload flow today is:

1. user opens [src/features/files/components/upload-file-modal.tsx](../src/features/files/components/upload-file-modal.tsx)
2. client validates selected files against [src/config/upload.ts](../src/config/upload.ts) and [src/config/input-limits.ts](../src/config/input-limits.ts)
3. client starts an upload session through `/api/file-uploads/sessions`
4. binary content is uploaded to managed R2 storage
5. client keeps local upload state until the user clicks **Upload Files**
6. `/api/file-uploads/commit` creates DB-backed file records
7. `FilesProvider` maps committed `FileDTO` records into deliverables for the project details screen

### Two Upload Families Exist

#### 1. Active UI Flow: upload session -> object upload -> commit

Main files:

- [src/features/files/lib/upload-client.ts](../src/features/files/lib/upload-client.ts)
- [src/features/files/components/upload-file-modal.tsx](../src/features/files/components/upload-file-modal.tsx)
- [src/features/files/providers/files-provider.tsx](../src/features/files/providers/files-provider.tsx)
- [src/app/api/file-uploads/sessions/route.ts](../src/app/api/file-uploads/sessions/route.ts)
- [src/app/api/file-uploads/content/route.ts](../src/app/api/file-uploads/content/route.ts)
- [src/app/api/file-uploads/multipart/parts/[partNumber]/route.ts](<../src/app/api/file-uploads/multipart/parts/[partNumber]/route.ts>)
- [src/app/api/file-uploads/multipart/complete/route.ts](../src/app/api/file-uploads/multipart/complete/route.ts)
- [src/app/api/file-uploads/multipart/abort/route.ts](../src/app/api/file-uploads/multipart/abort/route.ts)
- [src/app/api/file-uploads/object/route.ts](../src/app/api/file-uploads/object/route.ts)
- [src/app/api/file-uploads/commit/route.ts](../src/app/api/file-uploads/commit/route.ts)

This is the flow used by the upload modal in the current UI.

#### 2. Record-First Flow: create file row -> upload by file id

Main files:

- [src/app/api/files/route.ts](../src/app/api/files/route.ts)
- [src/app/api/files/[id]/content/route.ts](<../src/app/api/files/[id]/content/route.ts>)
- [src/app/api/files/[id]/multipart/route.ts](<../src/app/api/files/[id]/multipart/route.ts>)
- [src/app/api/files/[id]/multipart/parts/[partNumber]/route.ts](<../src/app/api/files/[id]/multipart/parts/[partNumber]/route.ts>)
- [src/app/api/files/[id]/multipart/complete/route.ts](<../src/app/api/files/[id]/multipart/complete/route.ts>)
- [src/app/api/files/[id]/multipart/abort/route.ts](<../src/app/api/files/[id]/multipart/abort/route.ts>)

Current implementation appears to keep these endpoints available, but I did not find the upload modal calling them.

### What Lives Where

#### File metadata

File metadata lives in PostgreSQL in the `mitfloww.files` table defined in [src/lib/db/schema/files.ts](../src/lib/db/schema/files.ts).

Important stored fields include:

- `projectId`
- `name`
- `originalName`
- `mimeType`
- `extension`
- `sizeBytes`
- `storageBucket`
- `storageKey`
- `previewEnabled`
- `paymentLocked`
- `paymentStatus`
- `revisionLimit`
- `extraRevisionCostCents`
- `priceCents`
- `uploadStatus`

#### Binary content

Binary content lives in Cloudflare R2 through [src/lib/storage/r2.ts](../src/lib/storage/r2.ts).

### How Final Storage Keys Are Created

Managed storage keys are created in [src/lib/uploads/final-storage-keys.ts](../src/lib/uploads/final-storage-keys.ts).

Current format is effectively:

```text
admin/<project-slug>/<timestamp-random>/<safe-file-name.ext>
```

Important details:

- prefix is `admin`
- project segment is slugified
- file name is normalized to ASCII-like safe characters
- a timestamp + random suffix is added so keys are unique

In the active upload-session flow, the key is created up front when the session is initiated.

In the record-first `/api/files` flow, the service builds a key from `projectId` or `"unassigned"` when the DB row is created.

### How Upload Sessions Are Tracked

Upload session DTOs are defined in [src/lib/dto/file-upload-sessions.ts](../src/lib/dto/file-upload-sessions.ts).

Important fields:

- `bucket`
- `localFileId`
- `mode`
- `multipartThresholdBytes`
- `partSizeBytes`
- `storageKey`
- `totalParts`
- `uploadId`
- `uploadSessionId`

Current implementation appears to track sessions:

- in React state inside the upload modal
- with an `uploadSessionId` generated once per modal batch
- with R2 multipart `uploadId` for large uploads

I did not find an `upload_sessions` DB table, so sessions are not durably persisted in the app database today.

### Normal Upload Flow for Small Files

Small file logic is mostly in [src/features/files/lib/upload-client.ts](../src/features/files/lib/upload-client.ts).

Flow:

1. `uploadFileToStorage()` posts to `/api/file-uploads/sessions`
2. server validates metadata using [src/lib/validation/files.ts](../src/lib/validation/files.ts)
3. `fileService.initiateUploadSession()` decides `mode: "single"` or `mode: "multipart"`
4. for `single`, the client `PUT`s file bytes to `/api/file-uploads/content?bucket=...&sizeBytes=...&storageKey=...`
5. `fileService.uploadUploadedObjectContent()` writes bytes to R2
6. client marks file status as `ready`
7. when the user commits, `FilesProvider.createFiles()` posts metadata to `/api/file-uploads/commit`
8. `fileService.commitUploadedFiles()` verifies the R2 object exists, then creates the DB row

### Multipart Upload Flow for Large Files

Multipart rules live in:

- [src/lib/uploads/multipart.ts](../src/lib/uploads/multipart.ts)
- [src/config/upload.ts](../src/config/upload.ts)

Current config:

- multipart threshold: `50 MB`
- part size: `25 MB`
- retry per part: `3`

Current browser flow:

1. client requests `/api/file-uploads/sessions`
2. if file size is above threshold, service creates an R2 multipart upload and returns `uploadId`
3. for each part, client calculates byte range with `getMultipartPartRange()`
4. client calls `GET /api/file-uploads/multipart/parts/[partNumber]` to get a presigned part URL
5. client uploads that part directly to R2 using the presigned URL
6. client captures the returned `ETag`
7. after all parts upload, client calls `POST /api/file-uploads/multipart/complete`
8. service validates parts are complete and sequential, then completes the R2 multipart upload
9. client marks file status as `ready`
10. user still must click **Upload Files** to commit DB records

Important detail: current implementation uploads parts sequentially in a `for` loop. `uploadConfig.multipartUploadConcurrency` exists, but I did not find it used in the client yet. TODO: confirm whether concurrent part uploads are planned.

Important detail: the code supports per-part retry, but comments in [src/features/files/lib/upload-client.ts](../src/features/files/lib/upload-client.ts) and [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) make it clear this is not a full refresh-resume implementation.

### Commit Semantics

Commit is the step that turns uploaded storage objects into real file records.

Client side:

- [src/features/files/providers/files-provider.tsx](../src/features/files/providers/files-provider.tsx)

Server side:

- [src/app/api/file-uploads/commit/route.ts](../src/app/api/file-uploads/commit/route.ts)
- [src/lib/services/file-service.ts](../src/lib/services/file-service.ts)

When commit runs, the client sends:

- project id
- currency
- batch revision settings
- payment lock / preview settings
- per-file price/name/size/mime/storage key metadata

The service then:

- validates the payload
- confirms `projectId` exists in the request payload
- confirms the storage key uses the managed upload prefix
- checks whether a DB record already exists for that storage key
- checks the R2 object exists and matches expected size
- creates one DB row per file
- returns both `committed` and `failed` arrays

This is why a file can be:

- uploaded to R2
- ready in the modal
- but not yet visible as a deliverable until commit succeeds

### Finalize Semantics

There is a `src/app/api/file-uploads/finalize` folder in the structure, but there is no `route.ts`.

Today:

- `commit` is implemented
- `finalize` is not implemented

So there is no separate finalize step beyond commit in the current codebase.

### Temp Upload Flow

The structure contains:

- `src/app/api/file-uploads/temp/content`
- `src/app/api/file-uploads/temp/multipart/abort`
- `src/app/api/file-uploads/temp/multipart/complete`
- `src/app/api/file-uploads/temp/multipart/parts/[partNumber]`

But no `route.ts` files are present there.

So the temp-upload route family is currently a placeholder, not a working implementation.

### Cancel, Delete, Abort, Retry, and Discard Behavior

#### Cancel while uploading

Upload modal logic:

- [src/features/files/components/upload-file-modal.tsx](../src/features/files/components/upload-file-modal.tsx)

When the user cancels an in-flight upload:

- the client aborts the active `AbortController`
- if the upload already produced an uploaded object/session, it tries to delete it through `/api/file-uploads/object`
- if cleanup fails, the upload is canceled locally and the cleanup is queued for retry

#### Delete after upload but before commit

If a file is already uploaded to storage but still only exists in the modal:

- deleting it calls `cleanupLocalFile()`
- that calls `/api/file-uploads/object`
- on failure, it queues retry in localStorage and warns the user via toast

#### Retry

Retry logic:

- cleans up previous uploaded object if present
- resets local file state to queued
- starts the upload again

#### Discard / close modal with unsaved uploads

Guard logic:

- [src/features/files/hooks/use-unsaved-upload-guard.ts](../src/features/files/hooks/use-unsaved-upload-guard.ts)

Behavior:

- warns on browser refresh via `beforeunload`
- intercepts back navigation with a history sentinel
- shows different confirmation copy depending on whether files are selected, uploading, or uploaded-but-not-committed
- if confirmed, tries to clean up all uploaded objects before closing

#### Multipart abort behavior

Server-side multipart abort logic exists in both route families:

- record-first: `/api/files/[id]/multipart/abort`
- upload-session: `/api/file-uploads/multipart/abort`

Abort reason is validated as either:

- `canceled`
- `failed`

In the record-first file flow, abort reason also influences the saved `uploadStatus`.

### Deferred Cleanup and Pending Upload Cleanup Queue

Deferred cleanup lives in:

- [src/features/files/lib/pending-upload-cleanup.ts](../src/features/files/lib/pending-upload-cleanup.ts)

If storage cleanup fails, the client stores a retry record in localStorage under:

```text
mitfloww.pending-upload-cleanups
```

That queue stores:

- bucket
- storage key
- upload id
- reason
- retry count
- last error

When the upload modal opens, it calls `retryPendingUploadCleanups({ limit: 5 })`.

This means:

- cleanup is best-effort in the moment
- cleanup may finish later on the next modal open
- browser localStorage is part of the client-side safety story

### Orphan Cleanup and Admin Recovery

Server-side orphan cleanup is in:

- [src/app/api/admin/file-uploads/orphans/route.ts](../src/app/api/admin/file-uploads/orphans/route.ts)
- [src/lib/services/file-service.ts](../src/lib/services/file-service.ts)

This scans R2 for managed upload keys that:

- are older than a threshold
- have no matching DB file record

It checks:

- stored objects
- in-progress multipart uploads

Important safety details:

- `dryRun` defaults to `true`
- `olderThanHours` defaults to `24`
- the service includes TODO comments saying auth should protect this endpoint before production use
- the service also includes a TODO saying scheduled infrastructure should run this automatically later

This is the server-side recovery path for abandoned browser uploads or failed cleanup.

### Validation Rules and Size/Type Limits

Config files:

- [src/config/upload.ts](../src/config/upload.ts)
- [src/config/input-limits.ts](../src/config/input-limits.ts)

Supported upload categories and limits:

- image: up to `100 MB`
- video: up to `5000 MB`
- archive (`.zip`): up to `5000 MB`
- document (`.pdf`): up to `5000 MB`

Accepted extensions currently include:

- `.jpg`, `.jpeg`, `.png`, `.webp`, `.avif`, `.svg`
- `.mp4`, `.mov`, `.webm`, `.mkv`
- `.zip`
- `.pdf`

Other important upload limits:

- max files per batch: `20`
- total batch size max: currently equal to `maxSingleUploadFileSizeBytes`
- project title limit: `80`
- file title limit: `120`
- effective max revision limit: `20`

Validation is enforced in more than one place on purpose:

- browser-side selection rules in `upload-file-modal.tsx`
- shared config in `upload.ts` and `input-limits.ts`
- server-side schema validation in [src/lib/validation/files.ts](../src/lib/validation/files.ts)
- submission-level business validation in [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts)

### Upload-Submission Validation

Upload-submission validation is centralized in:

- [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts)

It validates:

- `revisionLimit`
- `extraRevisionCost`
- per-file amount
- currency-aware minimums for file amounts and revision amounts

Examples:

- if `revisionLimit > 0`, extra revision cost becomes required
- file amounts must be at least the configured minimum for that currency
- revision limit must be a whole number and within plan/config limits

Current config examples:

- minimum file amount: `USD 10`, `INR 500`
- minimum extra revision amount: `USD 2`, `INR 10`

### Revision Settings

Relevant files:

- [src/features/files/components/revision-section.tsx](../src/features/files/components/revision-section.tsx)
- [src/features/files/components/revision-settings.ts](../src/features/files/components/revision-settings.ts)
- [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts)

Important detail: [src/features/files/components/revision-settings.ts](../src/features/files/components/revision-settings.ts) is just a re-export of shared validation helpers from `src/lib/validation/upload-submission.ts`.

That means revision behavior is already centralized in the validation layer. The feature file is a thin convenience export.

Revision settings are collected once per upload batch:

- `revisionLimit`
- `extraRevisionCost`

Those batch values are then written onto every committed file record in the batch.

### Deliverable Cards and Revision Data

Current deliverable UI files:

- [src/features/files/components/deliverable-card.tsx](../src/features/files/components/deliverable-card.tsx)
- [src/features/files/components/revision-section.tsx](../src/features/files/components/revision-section.tsx)
- [src/features/files/providers/files-provider.tsx](../src/features/files/providers/files-provider.tsx)
- [src/types/deliverables.ts](../src/types/deliverables.ts)

Current implementation appears to work like this:

- a deliverable is basically a UI projection of a committed `FileDTO`
- `FilesProvider.toDeliverable()` maps file DTOs into card-friendly shape
- the card shows:
  - name
  - size
  - created date
  - price
  - preview badge
  - payment badge
- revision settings are stored on the file row, but they are not currently displayed on the deliverable card

Related note: `Deliverable` has an optional `thumbnailUrl`, but `FilesProvider` does not currently populate it. So image files use the file content URL, and non-image previews may fall back to placeholder behavior.

### Payment Lock and Preview Mode Behavior

The upload modal sends:

- `paymentLocked`
- derived `paymentStatus`
- `previewEnabled`

Those values are stored in the DB.

Current implementation appears to use them mainly for:

- UI badges on deliverable cards
- filtering/counting metadata later if needed

Important practical note: [src/lib/services/file-service.ts](../src/lib/services/file-service.ts) serves `/api/files/[id]/content` based on `uploadStatus === uploaded`. I did not find server-side enforcement of `paymentLocked` or `previewEnabled` there. TODO: confirm whether preview/payment gating is planned elsewhere.

### Client-Side Responsibilities

The client upload layer is responsible for:

- validating selected files before upload
- keeping per-file local upload state
- creating one modal-level upload session id
- calculating multipart part ranges
- showing progress, retry, cancel, and delete UI
- collecting per-file price and batch revision/payment/preview settings
- committing uploaded objects into DB records
- warning before discarding unsaved uploads
- retrying failed storage cleanup from localStorage

### Server-Side Responsibilities

The server upload layer is responsible for:

- validating all route params, query values, and bodies
- deciding small vs multipart mode
- generating managed storage keys
- initiating/aborting/completing multipart uploads
- uploading small-file content to R2
- deleting uploaded managed objects
- verifying uploaded object existence before commit
- creating file DB rows
- scanning for orphaned uploads

### Upload Sequence Diagram

```mermaid
sequenceDiagram
    participant U as User
    participant M as Upload Modal
    participant API as /api/file-uploads/*
    participant S as FileService
    participant R2 as Cloudflare R2
    participant DB as PostgreSQL

    U->>M: Select files
    M->>API: POST /sessions
    API->>S: validate + initiateUploadSession
    S-->>API: bucket, storageKey, mode, uploadId
    API-->>M: UploadSessionDTO

    alt Single upload
        M->>API: PUT /content?bucket&sizeBytes&storageKey
        API->>S: uploadUploadedObjectContent
        S->>R2: PutObject
        R2-->>S: stored
        S-->>API: UploadedObjectDTO
        API-->>M: object ready
    else Multipart upload
        loop each part
            M->>API: GET /multipart/parts/[partNumber]
            API->>S: getPresignedMultipartPartUrl
            S-->>API: presigned URL
            API-->>M: URL + expiresAt
            M->>R2: PUT part directly
            R2-->>M: ETag
        end
        M->>API: POST /multipart/complete
        API->>S: completeUploadSessionMultipart
        S->>R2: CompleteMultipartUpload
        R2-->>S: completed
        S-->>API: UploadedObjectDTO
        API-->>M: object ready
    end

    U->>M: Click Upload Files
    M->>API: POST /commit
    API->>S: commitUploadedFiles
    S->>R2: HeadObject
    S->>DB: insert file rows
    DB-->>S: File records
    S-->>API: committed + failed
    API-->>M: Commit result
```

<a id="projects-and-deliverables-flow"></a>
## Projects and Deliverables Flow

### Projects List Page

Main files:

- [src/app/(dashboard)/projects/page.tsx](<../src/app/(dashboard)/projects/page.tsx>)
- [src/features/projects/components/projects-page.client.tsx](../src/features/projects/components/projects-page.client.tsx)
- [src/features/projects/providers/projects-provider.tsx](../src/features/projects/providers/projects-provider.tsx)

Current behavior:

- the page itself is thin
- `ProjectsPageClient` renders the UI
- projects live in client state in `ProjectsProvider`
- the provider starts with two seeded sample projects:
  - `brand-identity-v2`
  - `q3-marketing-site`
- tab filtering is local (`all`, `active`, `completed`)
- file counts and paid counts are hydrated from `/api/projects/metrics`

Practical takeaway: this is not a DB-backed project module yet. Current implementation appears to use local/in-memory projects plus DB-backed file metrics.

### Create Project Modal

Main file:

- [src/features/projects/components/create-project-modal.tsx](../src/features/projects/components/create-project-modal.tsx)

The modal collects:

- project name
- `paymentLockEnabled`
- `previewModeEnabled`

Important practical note: `ProjectsProvider.createProject()` currently only uses `name`. It creates a client-only project object and ignores `paymentLockEnabled` and `previewModeEnabled`.

So today:

- project creation is not persisted to the database
- the two delivery-control toggles in the create-project modal do not appear to affect saved project behavior

TODO: confirm whether project-level delivery controls are planned but not wired yet.

### Project Cards

Main file:

- [src/features/projects/components/project-card.tsx](../src/features/projects/components/project-card.tsx)

Project cards show:

- project status
- title
- client name
- file count
- paid/total count
- updated label

They navigate client-side with `router.push('/projects/{id}')`.

### Project Details Page

Main files:

- [src/app/(dashboard)/projects/[id]/page.tsx](<../src/app/(dashboard)/projects/[id]/page.tsx>)
- [src/features/projects/components/project-details.client.tsx](../src/features/projects/components/project-details.client.tsx)

What it does:

- looks up the project from `ProjectsProvider`
- asks `FilesProvider` to load project files from `/api/files?projectId=...&uploadStatus=uploaded`
- calculates total/paid/pending amounts from loaded deliverables
- renders deliverable cards
- opens the upload modal

### Delivery Controls

Main file:

- [src/features/projects/components/delivery-controls-section.tsx](../src/features/projects/components/delivery-controls-section.tsx)

This section is used in the upload modal, not in the project details summary card.

It controls:

- `paymentLock`
- `previewMode`

Those values are sent in the upload commit payload and stored on file records.

### Deliverable Card

Main file:

- [src/features/files/components/deliverable-card.tsx](../src/features/files/components/deliverable-card.tsx)

This is the current deliverable UI.

It renders:

- preview/placeholder image area
- file type badge
- name
- size
- date
- price
- preview enabled/disabled badge
- paid/pending badge

### Revision Section

Main files:

- [src/features/files/components/revision-section.tsx](../src/features/files/components/revision-section.tsx)
- [src/features/files/components/revision-settings.ts](../src/features/files/components/revision-settings.ts)

This collects batch-level revision metadata during upload.

It does not currently appear on the deliverable card after upload.

### How Project Details Relate to Files / Deliverables / Revisions

Current implementation appears to model this as:

- project details page
  - owns the route and page composition
- files provider
  - fetches committed `FileDTO` records for a project
- deliverables
  - are mapped from `FileDTO`
- revision data
  - is stored on each file record
  - collected during upload
  - not currently surfaced in the deliverable card UI

So in practice, "deliverable" is not a separate DB table in this repo. It is a UI-friendly view over committed file records.

<a id="admin-flow"></a>
## Admin Flow

### What the Admin Page Is For

Main files:

- [src/app/admin/page.tsx](../src/app/admin/page.tsx)
- [src/features/admin/components/admin-dashboard.tsx](../src/features/admin/components/admin-dashboard.tsx)

The admin page appears to be a realtime monitor for an external processing worker.

It currently:

- opens a WebSocket to `ws://localhost:4001`
- displays job stats and job tables
- fetches job details through `/api/admin/job/[id]`
- fetches previews through `/api/admin/preview/[id]`
- triggers retry through `/api/admin/retry/[id]`

This is a different subsystem from the normal project/deliverable flow.

### DLQ Concept

`DLQ` usually means dead-letter queue.

This repo exposes:

- [src/app/api/admin/dlq/route.ts](../src/app/api/admin/dlq/route.ts)

Current implementation appears to proxy an external worker endpoint, but I did not find a UI in this repo actively using the DLQ route.

### Retry Job Endpoint

- route: `/api/admin/retry/[id]`
- file: [src/app/api/admin/retry/[id]/route.ts](<../src/app/api/admin/retry/[id]/route.ts>)

Important note: it uses `GET` even though it appears to trigger a state change.

### Preview Endpoint

- route: `/api/admin/preview/[id]`
- file: [src/app/api/admin/preview/[id]/route.ts](<../src/app/api/admin/preview/[id]/route.ts>)

The admin UI polls preview up to 10 times with a 1-second delay between attempts.

For video previews:

- the UI uses `hls.js` for non-`.mp4` streams

### Job Status Endpoint

- route: `/api/admin/job/[id]`
- file: [src/app/api/admin/job/[id]/route.ts](<../src/app/api/admin/job/[id]/route.ts>)

The admin modal uses it to load logs and metadata for one job.

### Orphan Upload Cleanup Endpoint

- route: `/api/admin/file-uploads/orphans`
- file: [src/app/api/admin/file-uploads/orphans/route.ts](../src/app/api/admin/file-uploads/orphans/route.ts)

This is the only admin route family that works directly with this app's own storage and DB layers.

### Safe Rules for Touching Admin APIs

- Treat `/api/admin/*` worker proxy routes as contracts with an external service.
- Do not assume they follow the same response wrapper as file/upload routes.
- Do not silently change `GET`/`POST` semantics without checking the external worker.
- Expect local development to need services on `localhost:4000` and `localhost:4001`.
- Add auth before exposing worker/admin routes in a production environment.
- Prefer `dryRun=true` first when using orphan cleanup.

<a id="health-checks"></a>
## Health Checks

### `/health` Page

Main files:

- [src/app/health/page.tsx](../src/app/health/page.tsx)
- [src/features/health/components/health-check-page.tsx](../src/features/health/components/health-check-page.tsx)

This page is a manual diagnostics screen.

It has buttons to call:

- `/api/health`
- `/api/health/db`

It prints the raw JSON responses in the UI.

### `/api/health`

Main file:

- [src/app/api/health/route.ts](../src/app/api/health/route.ts)

What it does:

- runs `SELECT 1`
- returns DB connection status and schema name

### `/api/health/db`

Main file:

- [src/app/api/health/db/route.ts](../src/app/api/health/db/route.ts)

What it does:

- inserts one `health_checks` row
- reads it back
- deletes it

This is a more meaningful test than the simple DB ping because it exercises write and delete behavior.

### How to Use These During Local Dev or Deployment Validation

- Use `/health` first to confirm the app is up and can reach the database.
- Use `/api/health` or the Ping DB button to confirm base connectivity.
- Use `/api/health/db` or the CRUD button to confirm reads and writes in the `mitfloww` schema.
- If uploads fail, these health checks will not confirm R2 configuration. They only confirm app + DB health.

<a id="database-layer"></a>
## Database Layer

### DB Client Creation

Main file:

- [src/lib/db/client.ts](../src/lib/db/client.ts)

Behavior:

- uses `pg` `Pool`
- reads `DATABASE_URL`
- enables SSL with `rejectUnauthorized: false` in production
- caches the pool on `globalThis` outside production to avoid creating too many connections during dev reloads

### Schema Structure

Main files:

- [src/lib/db/schema/index.ts](../src/lib/db/schema/index.ts)
- [src/lib/db/schema/files.ts](../src/lib/db/schema/files.ts)
- [src/lib/db/schema/health.ts](../src/lib/db/schema/health.ts)

The app uses PostgreSQL schema `mitfloww`.

`files` table highlights:

- UUID primary key
- project/file naming fields
- storage bucket/key fields
- payment + preview flags
- revision fields
- price fields
- upload status
- soft-delete field `deletedAt`
- created/updated timestamps

Important DB details:

- upload/payment statuses are stored as numeric DB values but mapped to typed string enums in code
- `storageKey` has a unique index
- repository queries filter out soft-deleted rows by default

### Repository Pattern

Main file:

- [src/lib/repositories/file-repository.ts](../src/lib/repositories/file-repository.ts)

The repository owns:

- create
- findById
- findByStorageKey
- findMany
- softDelete
- update

This is the right place for query composition, filters, sorting, and deleted-row behavior.

### Service Pattern

Main file:

- [src/lib/services/file-service.ts](../src/lib/services/file-service.ts)

The service owns:

- file business rules
- storage coordination
- multipart orchestration
- commit/orphan cleanup
- DB + R2 combined behavior

### Practical Rule

If a service/repository already exists, do not put complex DB logic directly into:

- `page.tsx`
- Client Components
- route handlers

Keep route handlers thin and delegate.

### How to Add New DB-Backed Functionality in This Style

1. Add or extend schema in `src/lib/db/schema/*`.
2. Generate/update migrations in the `drizzle` folder using the team's migration workflow.
3. Add repository methods for the query/update behavior.
4. Add service methods for business rules and cross-resource logic.
5. Add validation schemas for inputs.
6. Add or update route handlers to parse input and call the service.
7. Keep UI code talking to the API, not directly to DB modules.

<a id="api-response-and-error-handling"></a>
## API Response and Error Handling

### Standard API Response Format

Main files:

- [src/lib/api/route.ts](../src/lib/api/route.ts)
- [src/lib/dto/api.ts](../src/lib/dto/api.ts)

Standard success format:

```ts
{
  data: ...,
  meta?: ...
}
```

Standard error format:

```ts
{
  error: {
    code: string,
    message: string,
    details?: ...
  }
}
```

### How `AppError` Works

Main file:

- [src/lib/errors/app-error.ts](../src/lib/errors/app-error.ts)

`AppError` carries:

- `message`
- `statusCode`
- `code`
- optional `details`

Common subclasses:

- `ValidationAppError` -> 400
- `NotFoundAppError` -> 404

### How Validation Errors Are Returned

`parseWithSchema()` in [src/lib/api/route.ts](../src/lib/api/route.ts):

- runs Zod parsing
- converts `ZodError` into `ValidationAppError`
- returns `validation_error` with issue details

### How Unexpected Errors Are Handled

If an error is not recognized:

- `errorResponse()` logs it with `console.error`
- returns HTTP 500
- response code is `internal_server_error`

### Status Codes You Will See in This Codebase

- `200` for normal success
- `201` for created resources or initiated sessions
- `400` for validation errors
- `404` for not found
- `409` for conflict/state problems
- `499` for request cancellation/abort
- `500` for unexpected failures
- `503` for missing storage configuration or unsupported presigning

### Important Exceptions to the Wrapper Rule

The following route families do not follow the standard wrapper:

- `/api/health*`
- `/api/admin*`

That is worth remembering when you consume these routes from client code.

<a id="validation"></a>
## Validation

### Where Validation Belongs

Main files:

- [src/lib/validation/files.ts](../src/lib/validation/files.ts)
- [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts)
- [src/config/upload.ts](../src/config/upload.ts)
- [src/config/input-limits.ts](../src/config/input-limits.ts)

Use this rule:

- shape/type/route/query/body validation belongs in `src/lib/validation/*`
- shared numeric/string/limit constants belong in `src/config/*`
- business validation that spans fields belongs in service or shared validation helpers

### Input Validation vs Business Validation

Examples of input validation:

- `id` must be a UUID
- `partNumber` must be a positive integer
- `mimeType` must be allowed
- `priceCents` must be a non-negative integer

Examples of business validation:

- multipart parts must be complete and sequential
- uploaded object must exist in storage before commit
- uploaded object size must match metadata
- revision cost becomes required when revision limit is greater than zero

### Upload Validation

Upload validation in [src/lib/validation/files.ts](../src/lib/validation/files.ts) covers:

- file metadata
- file creation/update payloads
- list query filters
- multipart query/body payloads
- upload session init
- commit payloads
- orphan cleanup payloads

### Upload-Submission Validation

Submission-level validation in [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts) covers:

- revision limit rules
- extra revision cost rules
- per-file price rules

### Route Param Validation

Examples:

- `fileIdParamsSchema`
- `multipartUploadPartParamsSchema`

Use these instead of parsing route params manually in handlers.

### Query / Pagination / Search / Sorting Validation

`fileQueryParamsSchema` handles:

- `page`
- `limit`
- `sort`
- `order`
- `search`
- `projectId`
- `paymentLocked`
- `paymentStatus`
- `previewEnabled`
- `uploadStatus`
- `includeTotal`

### Rules for Adding New Validation

- Put route-level schemas in `src/lib/validation/*`.
- Reuse constants from `src/config/*`.
- Prefer one shared schema over repeated inline checks.
- Keep route handlers thin by calling `parseWithSchema()`.
- Put cross-field validation in `.refine()` / `.superRefine()` or service rules.
- Do not duplicate upload limits or currency limits in component code.

<a id="i18n-and-localization"></a>
## i18n and Localization

### Where Messages Live

Main files:

- [src/i18n/messages/en.ts](../src/i18n/messages/en.ts)
- [src/i18n/messages/hi.ts](../src/i18n/messages/hi.ts)
- [src/i18n/messages/index.ts](../src/i18n/messages/index.ts)

The message shape is defined by `AppMessages` in `en.ts`.

### How Locale Is Resolved

Main files:

- [src/i18n/config.ts](../src/i18n/config.ts)
- [src/i18n/server.ts](../src/i18n/server.ts)

Server locale resolution checks:

1. custom header `x-app-locale`
2. cookie `mitfloww-locale`
3. `Accept-Language`
4. fallback to `en`

Practical note: I did not find middleware setting `x-app-locale` today, so cookie and browser language are likely the real inputs in current use.

### How to Use i18n in Components

Client side:

- use `useTranslations("namespace")` from [src/i18n/provider.tsx](../src/i18n/provider.tsx)

Server side:

- use `getRequestLocale()` from [src/i18n/server.ts](../src/i18n/server.ts)
- use `getMessages(locale)` from [src/i18n/messages/index.ts](../src/i18n/messages/index.ts)

### How to Add a New Text Label

1. Add the key to the correct namespace in [src/i18n/messages/en.ts](../src/i18n/messages/en.ts).
2. Add the matching translation in [src/i18n/messages/hi.ts](../src/i18n/messages/hi.ts).
3. Consume it through `useTranslations()` or server-side message lookup.

### How Locale Switching Works

Main files:

- [src/i18n/use-locale-switch.ts](../src/i18n/use-locale-switch.ts)
- [src/components/layout/locale-switcher.tsx](../src/components/layout/locale-switcher.tsx)

Behavior:

- sets locale cookie on the client
- uses `router.replace()` or `router.refresh()`
- does not add locale prefixes to URLs

Important note: `LocaleSwitcher` exists, but I did not find it mounted in the current app layout or navbar.

### Practical Rule

Avoid hardcoded user-facing text if the i18n pattern is already available.

Current codebase still has some hardcoded English strings in a few places, especially around upload-confirm dialogs and admin proxy behavior, so treat i18n as the preferred direction even if older code is mixed.

<a id="ui-and-component-system"></a>
## UI and Component System

### Shared UI

Main folder:

- `src/components/ui`

Examples:

- [src/components/ui/button.tsx](../src/components/ui/button.tsx)
- [src/components/ui/card.tsx](../src/components/ui/card.tsx)
- [src/components/ui/input.tsx](../src/components/ui/input.tsx)
- [src/components/ui/dialog.tsx](../src/components/ui/dialog.tsx)
- [src/components/ui/modal.tsx](../src/components/ui/modal.tsx)
- [src/components/ui/confirm-action-dialog.tsx](../src/components/ui/confirm-action-dialog.tsx)
- [src/components/ui/status-badge.tsx](../src/components/ui/status-badge.tsx)
- [src/components/ui/file-type-icon-badge.tsx](../src/components/ui/file-type-icon-badge.tsx)
- [src/components/ui/toast.tsx](../src/components/ui/toast.tsx)
- [src/components/ui/toaster.tsx](../src/components/ui/toaster.tsx)

### Layout Components

Main folder:

- `src/components/layout`

Examples:

- [src/components/layout/app-navbar.tsx](../src/components/layout/app-navbar.tsx)
- [src/components/layout/locale-switcher.tsx](../src/components/layout/locale-switcher.tsx)

### Skeleton Components

Main folder:

- `src/components/skeletons`

Examples:

- [src/components/skeletons/deliverable-card-skeleton.tsx](../src/components/skeletons/deliverable-card-skeleton.tsx)
- [src/components/skeletons/grid-skeleton.tsx](../src/components/skeletons/grid-skeleton.tsx)
- [src/components/skeletons/page-header-skeleton.tsx](../src/components/skeletons/page-header-skeleton.tsx)

### Feature Components

Feature-specific UI belongs under `src/features/<domain>/components`.

Examples:

- upload modal is feature-specific
- deliverable card is file-domain UI
- project card is project-domain UI

### When to Create Shared UI vs Feature-Specific UI

Create shared UI when:

- the component is generic
- it has no domain-specific language
- it could be reused across unrelated pages

Create feature UI when:

- it is tightly tied to a domain workflow
- it needs domain DTOs/provider hooks
- its behavior is specific to files, projects, admin, or health

### Loading States

This repo uses two loading patterns:

- route-level `loading.tsx`
- component-level loading state inside Client Components

Examples:

- `/projects` and `/projects/[id]` use route loading files
- `health-check-page.tsx` and `admin-dashboard.tsx` manage their own loading states

### Toast Usage

Main files:

- [src/hooks/use-toast.tsx](../src/hooks/use-toast.tsx)
- [src/components/ui/toaster.tsx](../src/components/ui/toaster.tsx)

Use toasts for:

- upload success/failure
- deferred cleanup warnings
- validation warnings

### Modal / Dialog / Confirm Action Usage

Use:

- `Dialog` / `Modal` for standard modal UI
- `ConfirmActionDialog` for risky flows like discard, delete, cancel

The confirm dialog already supports:

- variant styling
- loading state
- dismiss prevention
- attention animation

### Status Badge / File Type Icon Usage

- `StatusBadge` is the shared badge component for paid/pending/preview/project-state statuses.
- `FileTypeIconBadge` is the shared file-type icon wrapper for image/video/pdf markers.

<a id="state-management-and-providers"></a>
## State Management and Providers

### `LocalizationProvider`

File:

- [src/i18n/provider.tsx](../src/i18n/provider.tsx)

Owns:

- current locale
- loaded message dictionary

Mounted in:

- [src/app/layout.tsx](../src/app/layout.tsx)

### `ToastProvider`

File:

- [src/hooks/use-toast.tsx](../src/hooks/use-toast.tsx)

Owns:

- active toast list
- `toast()` API
- dismiss behavior

Mounted in:

- [src/app/layout.tsx](../src/app/layout.tsx)

### `ProjectsProvider`

File:

- [src/features/projects/providers/projects-provider.tsx](../src/features/projects/providers/projects-provider.tsx)

Owns:

- in-memory project list
- local `createProject()`
- `getProjectById()`
- metrics hydration from `/api/projects/metrics`

Mounted in:

- [src/app/(dashboard)/layout.tsx](<../src/app/(dashboard)/layout.tsx>)

### `FilesProvider`

File:

- [src/features/files/providers/files-provider.tsx](../src/features/files/providers/files-provider.tsx)

Owns:

- deliverables grouped by project id
- file loading status per project
- API call to load project files
- commit flow wrapper through `createFiles()`

Mounted in:

- [src/app/(dashboard)/layout.tsx](<../src/app/(dashboard)/layout.tsx>)

### When to Add a Provider vs Local State

Use local component state when:

- the state only matters inside one component tree
- it is mostly UI-only

Use a provider when:

- multiple child components need the same state/actions
- the state belongs to a route branch
- you want a shared cache or domain-specific actions across a page area

Do not add a provider just to avoid prop drilling once or twice.

<a id="query-helpers"></a>
## Query Helpers

Main files:

- [src/lib/query/files.ts](../src/lib/query/files.ts)
- [src/lib/query/pagination.ts](../src/lib/query/pagination.ts)
- [src/lib/query/search.ts](../src/lib/query/search.ts)
- [src/lib/query/sorting.ts](../src/lib/query/sorting.ts)

### What They Do

- `files.ts`
  - typed file list query/filter model
- `pagination.ts`
  - default page values
  - offset calculation
  - pagination metadata calculation
- `search.ts`
  - safe `LIKE` pattern building
- `sorting.ts`
  - allowed sort orders

### Where to Use Them

- route handlers should validate incoming query params
- repositories should use typed query objects
- shared query parsing should stay centralized

### Practical Rule

Do not duplicate search/sort/pagination parsing in route files if helpers already exist.

<a id="styling"></a>
## Styling

Main files:

- [src/app/globals.css](../src/app/globals.css)
- [src/lib/utils/cn.ts](../src/lib/utils/cn.ts)
- [src/types/styles.d.ts](../src/types/styles.d.ts)

### Styling Approach

This app uses Tailwind CSS v4 with theme tokens defined in `globals.css`.

You will see:

- token-based color names
- utility classes directly in components
- shared `cn()` helper to merge class names safely

### `cn()` Helper

File:

- [src/lib/utils/cn.ts](../src/lib/utils/cn.ts)

It combines:

- `clsx`
- `tailwind-merge`

Use it whenever variant or conditional classes are involved.

### Shared Styling Conventions

- use theme tokens from `globals.css`
- prefer existing button/card/input/dialog patterns before inventing new ones
- keep shared UI variants consistent with existing `variant` props
- use skeletons instead of empty flashes during async loads

### Keeping UI Consistent

- start with shared UI primitives from `components/ui`
- use feature components for domain composition
- reuse badge, dialog, modal, button, and card patterns before writing custom markup

<a id="middleware"></a>
## Middleware

No `middleware.ts` file is currently present at the repo root or under `src`.

If middleware is added later, the usual place would be:

- `middleware.ts` at the project root

Typical reasons to add it later:

- auth/session gatekeeping
- locale redirects or locale headers
- request logging
- route guards
- cross-cutting headers/cookies

Because there is no middleware today:

- locale is not URL-prefixed
- `x-app-locale` is not automatically injected by middleware
- admin/auth route protection is not enforced at middleware level

<a id="development-workflow"></a>
## Development Workflow

### Install Dependencies

The repo contains both:

- `package-lock.json`
- `pnpm-lock.yaml`

So the package manager convention is not completely clear from the repository alone. TODO: confirm whether the team standard is `npm` or `pnpm`.

### Environment Setup

Start from:

- [.env.example](../.env.example)

At minimum, local development for DB-backed pages needs `DATABASE_URL`.

If you want to test uploads end to end, you also need the R2 variables used by [src/lib/storage/r2.ts](../src/lib/storage/r2.ts).

### Available Package Scripts

From [package.json](../package.json):

- `dev` -> `next dev`
- `build` -> `next build`
- `start` -> `next start`
- `lint` -> `eslint`

### Practical Run / Verify Flow

1. Install dependencies using the team-approved package manager.
2. Create a local `.env` from [.env.example](../.env.example).
3. Make sure PostgreSQL is reachable at `DATABASE_URL`.
4. If testing uploads, configure R2 values too.
5. Run the dev script from `package.json`.
6. Open:
   - `/`
   - `/projects`
   - `/projects/brand-identity-v2`
   - `/projects/q3-marketing-site`
   - `/analytics`
   - `/team`
   - `/health`
   - `/admin`
7. Run the lint script.
8. Run the build script as the closest thing to a full-project validation step.

### Things Not Defined in `package.json`

I did not find scripts for:

- dedicated `typecheck`
- `test`

So today:

- `lint` is your lint check
- `build` is your best all-up verification step

### How to Verify Health

- Use `/health`
- Use the two buttons there
- Check raw responses from `/api/health` and `/api/health/db`

### How to Test File Upload

1. Open `/projects`.
2. Open one project details page.
3. Launch the upload modal.
4. Try a supported small file first.
5. Add price and revision settings.
6. Click **Upload Files**.
7. Confirm the file appears as a deliverable card after commit.
8. Try cancel/delete/retry to verify cleanup behavior.

Important note: without working R2 config, upload flows will fail with storage-related errors.

### How to Inspect Browser and Server Behavior

- Watch the browser network tab for `/api/file-uploads/*`, `/api/files`, and `/api/projects/metrics`.
- Watch the browser console for upload warnings.
- Watch the Next.js server logs for:
  - unhandled API errors
  - multipart upload warnings
  - cleanup warnings

### Admin-Specific Dev Note

The admin page also expects:

- HTTP worker on `localhost:4000`
- WebSocket feed on `localhost:4001`

Without those, admin APIs and realtime monitoring will not fully work.

<a id="environment-variables"></a>
## Environment Variables

The following variables are referenced in code today.

| Variable | Where used | Purpose | Required in local dev? | Safe example value |
| --- | --- | --- | --- | --- |
| `DATABASE_URL` | [src/lib/db/client.ts](../src/lib/db/client.ts), [drizzle.config.ts](../drizzle.config.ts) | PostgreSQL connection string for app runtime and Drizzle tooling | Yes, if you want DB-backed routes/pages to work | `postgresql://user:password@127.0.0.1:5432/mitfloww` |
| `R2_ACCOUNT_ID` | [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) | Cloudflare R2 account id for credential-based storage access | Required for end-to-end upload/storage in most local setups | `your-account-id` |
| `R2_ACCESS_KEY_ID` | [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) | Cloudflare R2 access key id | Required for end-to-end upload/storage in most local setups | `your-access-key-id` |
| `R2_SECRET_ACCESS_KEY` | [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) | Cloudflare R2 secret key | Required for end-to-end upload/storage in most local setups | `your-secret-access-key` |
| `R2_BUCKET_NAME` | [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) | Default storage bucket name | Required for end-to-end upload/storage in most local setups | `mitfloww-files` |
| `R2_PUBLIC_BASE_URL` | [src/lib/storage/r2.ts](../src/lib/storage/r2.ts) | Public base URL used for public file URLs or presign-related fallback behavior | Optional, but useful depending on deployment | `https://cdn.example.com` |
| `NODE_ENV` | [src/lib/db/client.ts](../src/lib/db/client.ts) | Controls dev pool reuse and production SSL behavior | Usually no; Next sets this automatically | `development` |

### Important Non-Env Configuration Note

These are hardcoded in code, not environment variables:

- admin worker HTTP base: `http://localhost:4000`
- admin worker WebSocket feed: `ws://localhost:4001`

If you need those to vary by environment, that would be a future refactor.

### Browser-Exposed Environment Variables

I did not find any `NEXT_PUBLIC_*` environment variable usage in the current codebase.

If you add one later, remember:

- only `NEXT_PUBLIC_*` variables are exposed to browser bundles
- server-only secrets must stay unprefixed and must not be imported into Client Components

<a id="how-to-add-a-new-feature"></a>
## How to Add a New Feature

### Add a New Page

1. Create a route file under `src/app/.../page.tsx`.
2. Keep the page thin. Compose feature components there.
3. If the route needs loading UI, add `loading.tsx`.
4. If the page is interactive, hand off to a Client Component under `src/features/<domain>/components`.

Example pattern from this repo:

- page file: [src/app/(dashboard)/projects/page.tsx](<../src/app/(dashboard)/projects/page.tsx>)
- client feature file: [src/features/projects/components/projects-page.client.tsx](../src/features/projects/components/projects-page.client.tsx)

### Add a New Feature Component

1. Put domain-specific UI under `src/features/<domain>/components`.
2. Add `"use client"` only if you need browser hooks or APIs.
3. Pull shared primitives from `src/components/ui`.
4. Keep API calls and provider usage close to the feature, not inside shared UI primitives.

### Add a New API Route

1. Add `route.ts` under `src/app/api/...`.
2. Parse route/query/body input with `parseWithSchema()`.
3. Use `successResponse()` / `errorResponse()` if the route follows the standard pattern.
4. Keep the route thin and delegate to a service.

### Add a New DB-Backed Operation

1. Extend schema in `src/lib/db/schema/*` if needed.
2. Add repository logic in `src/lib/repositories/*`.
3. Add service logic in `src/lib/services/*`.
4. Add DTOs/mappers if the API shape needs them.
5. Add route validation schemas.
6. Call the service from the route handler.

### Add a New Validation Rule

1. Put shared rule logic in `src/lib/validation/*`.
2. Put reusable constants in `src/config/*`.
3. Reuse that rule from both UI and API where appropriate.
4. Avoid embedding magic numbers in components.

### Add a Shared UI Component

1. Put it in `src/components/ui`.
2. Accept props for variants/states instead of hardcoding one use case.
3. Reuse `cn()` for conditional classes.
4. Keep domain-specific wording or DTO types out of it.

### Add a Localized Message

1. Add the key in `en.ts`.
2. Add the matching key in `hi.ts`.
3. Read it via `useTranslations()` or server message lookup.
4. Avoid hardcoded user-facing strings unless you are touching an area that has not been localized yet.

### Add a Loading Skeleton

1. Create a skeleton component under `src/components/skeletons` or `src/features/<domain>/skeletons`.
2. Reference it from `loading.tsx` if it is route-level.
3. Use component-level loading state if the route itself does not suspend.

<a id="how-to-modify-existing-file-upload-behavior"></a>
## How to Modify Existing File Upload Behavior

### Where Client Upload Code Lives

- [src/features/files/components/upload-file-modal.tsx](../src/features/files/components/upload-file-modal.tsx)
- [src/features/files/components/upload-file-item.tsx](../src/features/files/components/upload-file-item.tsx)
- [src/features/files/lib/upload-client.ts](../src/features/files/lib/upload-client.ts)
- [src/features/files/lib/pending-upload-cleanup.ts](../src/features/files/lib/pending-upload-cleanup.ts)
- [src/features/files/hooks/use-unsaved-upload-guard.ts](../src/features/files/hooks/use-unsaved-upload-guard.ts)
- [src/features/files/providers/files-provider.tsx](../src/features/files/providers/files-provider.tsx)

### Where Upload API Routes Live

- `src/app/api/file-uploads/*` for the active upload-session/object/commit flow
- `src/app/api/files/[id]/*` for the record-first flow

### Where Validation Lives

- [src/lib/validation/files.ts](../src/lib/validation/files.ts)
- [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts)

### Where Submission-Level Validation Lives

- [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts)

If you need to change revision/file-amount rules, start there.

### Where Storage Logic Lives

- [src/lib/storage/r2.ts](../src/lib/storage/r2.ts)
- [src/lib/uploads/multipart.ts](../src/lib/uploads/multipart.ts)
- [src/lib/uploads/final-storage-keys.ts](../src/lib/uploads/final-storage-keys.ts)

### Where Repository/Service Logic Lives

- [src/lib/services/file-service.ts](../src/lib/services/file-service.ts)
- [src/lib/repositories/file-repository.ts](../src/lib/repositories/file-repository.ts)

### Where Revision Settings Live

- UI wrapper: [src/features/files/components/revision-settings.ts](../src/features/files/components/revision-settings.ts)
- actual shared rule logic: [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts)

### Practical Safe-Change Checklist

When changing upload behavior:

1. Check whether the change belongs to the active `file-uploads` flow, the record-first `files/[id]` flow, or both.
2. Update shared config/validation first.
3. Update client-side checks second.
4. Update server validation third.
5. Update service/storage behavior last.
6. Verify cancel, retry, discard, and cleanup still work.
7. Test both a small upload and a multipart upload.

### How to Avoid Breaking Multipart Upload

- Do not change part-size or threshold constants casually.
- Keep client and server using the same multipart rules from shared modules.
- Preserve `ETag` handling.
- Preserve sequential/complete part validation in the service.
- Preserve abort-on-failure cleanup logic.
- Preserve the managed storage key prefix rule.
- Preserve commit's storage existence/size checks.
- Remember that commit happens after upload. Do not accidentally treat "uploaded to storage" as "saved in DB".

<a id="architecture-rules"></a>
## Architecture Rules

- Keep route files thin.
- Put business logic in services.
- Put DB access in repositories.
- Put storage-specific logic in storage/upload modules.
- Put DTO mapping in DTO files.
- Put validation in validation/config files.
- Keep reusable UI in `components/ui`.
- Keep domain UI in `features/<domain>/components`.
- Avoid importing server-only modules into Client Components.
- Do not access DB/storage directly from Client Components.
- Do not duplicate upload limits or validation constants.
- Prefer typed DTOs from `src/types` and `src/lib/dto`.
- Keep `page.tsx` mostly as a composition layer.
- Use `loading.tsx` and skeletons for async route states.
- Use the existing toast/dialog/modal patterns.
- Keep revision behavior centralized in `revision-settings.ts` only as a re-export surface; the shared logic actually lives in `src/lib/validation/upload-submission.ts`.
- Keep upload-submission validation centralized in [src/lib/validation/upload-submission.ts](../src/lib/validation/upload-submission.ts).
- Treat `/api/files/...` and `/api/file-uploads/...` as separate flow families unless you intentionally unify them.
- Treat admin worker proxy routes as external contracts, not ordinary app-internal services.

<a id="common-pitfalls-for-net--angular-developers"></a>
## Common Pitfalls for .NET / Angular Developers

- Page routes are file-system based.
- `route.ts` is an API endpoint, not a React component.
- Client Components need `"use client"`.
- Server Components cannot use browser APIs or browser hooks.
- Client Components cannot directly import DB or storage modules.
- Environment variables exposed to the browser require a `NEXT_PUBLIC_` prefix.
- `(dashboard)` route groups do not appear in the URL.
- `[id]` is a dynamic route segment.
- `loading.tsx` is automatic route loading UI for that segment.
- API routes run server-side.
- Route handlers use `Request` / `Response` / `NextResponse` style, not ASP.NET controller classes.
- In this repo, project creation is currently client-memory based, not fully DB-backed.
- In this repo, "uploaded to storage" and "committed as a deliverable" are different states.

<a id="detailed-glossary"></a>
## Detailed Glossary

| Term | Meaning in this project |
| --- | --- |
| App Router | Next.js routing system under `src/app`. Routes come from folders and files, not from a central route config file. |
| Server Component | Default App Router component type. Runs on the server and can compose data-aware server logic, but cannot use browser hooks/APIs. |
| Client Component | React component marked with `"use client"`. Runs in the browser and can use hooks, events, file input, drag/drop, and router navigation. |
| Route Handler | A server endpoint implemented in `route.ts` under `src/app/api/...`. Comparable to a minimal API endpoint. |
| Layout | A shared wrapper for nested routes. In this repo, the dashboard layout mounts the navbar and domain providers. |
| Loading UI | A `loading.tsx` file that Next.js automatically shows while a route segment is loading. |
| Dynamic Route | A route segment like `[id]` whose value is passed via `params`. |
| Route Group | A folder like `(dashboard)` used for organization/layout grouping without affecting the public URL. |
| Provider | A React context owner that exposes shared state/actions. Examples: localization, toast, projects, files. |
| DTO | Data Transfer Object. Typed payload shape used between service, route, and client layers. |
| Repository | The DB access layer. In this repo, `DrizzleFileRepository` owns file-table queries and updates. |
| Service | The business logic layer. In this repo, `FileService` coordinates validation assumptions, DB state, and R2 storage behavior. |
| R2 | Cloudflare object storage used for uploaded file bytes. |
| Multipart Upload | Large-file upload split into numbered parts, each returning an `ETag`, then completed server-side. |
| Orphan Upload | A managed R2 object or multipart session that exists in storage but has no matching DB file record. |
| DLQ | Dead-letter queue. Usually means failed jobs/messages waiting for inspection or replay. This repo exposes a worker DLQ endpoint, but the UI usage is minimal. |
| Toast | Temporary notification UI for success/error/warning/info feedback. |
| Skeleton | Placeholder loading UI shaped like the final component. |
| Deliverable | Current UI term for a committed uploaded file record shown on the project details page. |
| Revision | File metadata for revision limit and extra revision cost, collected at upload time and stored with file records. |
| Upload Session | The client/server handshake result that gives the browser bucket/key/mode/uploadId details before bytes are uploaded. |
| Validation | Shape/type/rule checking applied in schemas and service rules before work is performed. |
| Middleware | A cross-cutting request layer that could run before routes. No middleware file exists today in this repo. |
| API Response Wrapper | The `{ data, meta? }` / `{ error }` shape returned by most file/upload/project APIs via `successResponse()` and `errorResponse()`. |
| Storage Key | The R2 object key, for example `admin/project-slug/timestamp/file.ext`. |
| DTO Mapper | Code that converts DB records into DTOs, such as `toFileDTO()` in [src/lib/dto/file-mappers.ts](../src/lib/dto/file-mappers.ts). |

<a id="final-onboarding-checklist"></a>
## Final Onboarding Checklist

- [ ] Run the app locally.
- [ ] Open `/`, `/projects`, `/analytics`, `/team`, `/health`, and `/admin`.
- [ ] Call `/api/health` and `/api/health/db`.
- [ ] Create, upload, cancel, retry, and delete a file if storage is configured.
- [ ] Trace one page flow from `page.tsx` to its feature component.
- [ ] Trace one API flow from `route.ts` to validation to service to repository/storage.
- [ ] Trace the file upload flow from upload modal to `/api/file-uploads/*` to `file-service.ts` to R2/DB.
- [ ] Make one small UI change using shared components.
- [ ] Make one small validation change in the right validation/config file.
- [ ] Read the architecture rules section before making larger changes.
