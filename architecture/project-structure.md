# Modular Feature Architecture

## 📚 Table of Contents

* [**Overview**](#overview)
* [**Example**](#example)

    * [Folder Structure](#folder-structure)
  * [**Architecture Breakdown**](#architecture-breakdown)

    * [app/ folder](#app-folder)
    * [modules/ folder](#modules-folder-the-heart)
    * [lib/ folder](#lib-folder)
    * [components/](#components)
    * [hooks/](#hooks)
  * [Key Principles](#key-principles)
* [**Problem**](#problem)
* [**Solution**](#solution)
* [**Notes**](#notes)

---

## Overview

Feature-based modular architecture
with service/repository layering
inside a Next.js App Router structure
inspired by Clean Architecture principles

---

## Problem

---

## Solution

---

## Example

### Folder Structure

```
/src
  /app
    /(public)
      page.tsx
      login
      signup
    /(dashboard)
      dashboard
      projects
      payments
    /api
      /auth
        route.ts
      /files
        upload
          route.ts
        preview
          route.ts
      /payments
        route.ts

  /components
    ui
    forms
    layout

  /modules
    auth
      auth.service.ts
      auth.repository.ts
      auth.types.ts
    files
      file.service.ts
      preview.service.ts
      watermark.service.ts
      file.repository.ts
    payments
      payment.service.ts
      escrow.service.ts

  /lib
    db
      client.ts
    storage
      r2.ts
    queue
      jobs.ts
    utils
      logger.ts
      validation.ts

  /hooks
    useAuth.ts
    useFiles.ts

  /types
    user.ts
    file.ts

  /config
    env.ts
```

---

## Architecture Breakdown

### app/ folder

This is the **Next.js router layer**.

It handles:

* pages
* layouts
* API endpoints

Example:

```
/app/api/files/upload/route.ts
```

This layer must remain **thin**.

```ts
export async function POST(req: Request) {
  return fileService.upload(req)
}
```

All business logic belongs to services.

---

### modules/ folder (the heart)

This is where **business logic lives**.

Example:

```
modules/files/file.service.ts
```

```ts
export async function uploadFile(data) {
  const file = await storage.upload(data)
  return fileRepository.save(file)
}
```

Key characteristics:

* Framework-independent
* Contains core application logic
* Easily portable to a Node.js backend

---

### lib/ folder

Infrastructure layer.

Examples:

Database connection:

```
lib/db/client.ts
```

Storage client:

```
lib/storage/r2.ts
```

Queue system:

```
lib/queue/jobs.ts
```

Think of it as:

> System plumbing (external integrations)

---

### components/

Reusable UI components.

```
components/ui/button.tsx
components/layout/navbar.tsx
```

Rules:

* No business logic
* Pure UI

---

### hooks/

React-specific logic.

Example:

```ts
useAuth()
useFileUpload()
```

Rules:

* UI interaction logic only
* No backend/business logic

---

## Key Principles

* Keep **API routes thin**
* Move all logic to **services**
* Keep **database logic in repositories**
* Separate **business logic from framework**
* Design for **future backend migration**

---

## Notes

* Avoid mixing business logic inside UI or API routes
* Keep modules self-contained by feature
* Prefer clarity over abstraction
* Structure should scale with features, not files

---

[⬆ Back to Table of Contents](#-table-of-contents)
