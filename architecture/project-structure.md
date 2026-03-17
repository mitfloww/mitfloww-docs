# Architecture Name - Modular Feature Architecture

## Overview
Feature-based modular architecture
with service/repository layering
inside a Next.js App Router structure
inspired by Clean Architecture principles

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

## Notes
### app/ folder

This is the Next router layer.

It handles:

* pages
* layouts
* API endpoints

Example:
```
/app/api/files/upload/route.ts
```
But this file should stay thin.

Example:
```ts
export async function POST(req: Request) {
  return fileService.upload(req)
}
```
All logic goes to services.

---
### modules/ folder (the heart)

This is where the business logic lives.

Example:
```
modules/files/file.service.ts
```
Example service:
```ts
export async function uploadFile(data) {
  const file = await storage.upload(data)
  return fileRepository.save(file)
}
```
This layer is framework independent.

Later you can move it into a Node backend easily.

---
### lib/ folder

Infrastructure code lives here.

Examples:

Database connection:
```
lib/db/client.ts
```
Storage client:
```
lib/storage/r2.ts
```
Queue system (future):
```
lib/queue/jobs.ts
```
Think of it as system plumbing.

---
### components/

Reusable UI pieces.
```
components/ui/button.tsx
components/layout/navbar.tsx
```
No business logic here.

---
### hooks/

React logic only.

Example:
```ts
useAuth()
useFileUpload()
```