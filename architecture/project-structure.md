# Feature-Based Frontend Architecture (Next.js App Router)

## 📚 Table of Contents

* [Overview](#overview)
* [Project Structure](#project-structure)
* [Architecture Breakdown](#architecture-breakdown)

  * [app/ (Routing Layer)](#app-routing-layer)
  * [features/ (Core Application Layer)](#features-core-application-layer)
  * [components/ (Shared UI)](#components-shared-ui)
  * [lib/ (Infrastructure Layer)](#lib-infrastructure-layer)
  * [i18n/ (Localization System)](#i18n-localization-system)
  * [types/](#types)
* [Usage Patterns](#usage-patterns)
* [End-to-End Flow](#end-to-end-flow)
* [Key Principles](#key-principles)
* [Notes](#notes)

---

## Overview

This project uses a **feature-based modular architecture** built on top of:

* Next.js App Router
* Tailwind v4 design tokens (`@theme`)
* shadcn-style UI primitives
* Scoped feature logic

This is **NOT backend Clean Architecture**.

Instead, it is optimized for:

* frontend scalability
* UI composition
* feature isolation
* maintainability

---

## Project Structure

```
src
├── app
│ ├── (dashboard)
│ │ ├── analytics
│ │ ├── projects
│ │ ├── team
│ │ ├── layout.tsx
│ │ └── page.tsx
│ ├── admin
│ ├── api
│ ├── health
│ ├── globals.css
│ └── layout.tsx

├── components
│ ├── layout
│ └── ui

├── features
│ ├── admin
│ ├── analytics
│ ├── dashboard
│ ├── health
│ ├── projects
│ └── team

├── i18n
├── lib
├── hooks
├── config
└── types
```

---

## Architecture Breakdown

---

### app/ (Routing Layer)

Handles:

* routes
* layouts
* server boundaries
* API endpoints

Example:

```
app/(dashboard)/projects/page.tsx
```

Rules:

* NO business logic
* ONLY composition
* imports feature components

```tsx
import ProjectsPageClient from "@/features/projects/components/projects-page.client";
export default function Page() {
  return <ProjectsPageClient />;
}
```

---

### features/ (Core Application Layer)

This is the **heart of the app**.

Each feature is self-contained.

Example:

```
features/projects/
  components/
  providers/
```

Contains:

* UI composition
* feature-specific state
* business logic (frontend level)

---

### components/ (Shared UI)

Reusable UI primitives.

```
components/ui/button.tsx
components/ui/dialog.tsx
components/layout/app-navbar.tsx
```

Rules:

* NO domain knowledge
* NO feature logic
* ONLY reusable UI

---

### lib/ (Infrastructure Layer)

External integrations and utilities.

Examples:

```
lib/db
lib/storage
lib/queue
lib/utils
```

Rules:

* No UI
* No feature logic
* Only system-level helpers

---

### i18n/ (Localization System)

Centralized localization.

```
i18n/messages/en.ts
i18n/messages/hi.ts
```

Rules:

* ALL user-facing text must come from i18n
* No hardcoded strings in UI

Usage:

```tsx
const t = useTranslations("projectsPage");
t("title");
```

Language switching:

* logic implemented
* no UI required yet

---

### types/

Global TypeScript definitions.

---

## Usage Patterns

---

### CASE 1: Adding a New Page

Example: `/billing`

```
app/(dashboard)/billing/page.tsx
features/billing/components/billing-page-content.tsx
```

Flow:

```
page.tsx → feature component
```

---

### CASE 2: Adding UI inside a Page

Example: Filters in projects

❌ Wrong:

```
app/
```

✅ Correct:

```
features/projects/components/project-filters.tsx
```

---

### CASE 3: Shared UI Component

Example:

```
components/ui/button.tsx
```

Rule:

> If it has zero domain knowledge → belongs in `components/ui`

---

### CASE 4: Shared Logic

Example:

```
lib/utils/*
```

---

### CASE 5: Feature-Specific Logic

Example:

```
features/projects/hooks/
features/projects/utils/
```

---

### CASE 6: Global State / Provider

Example:

```
features/projects/providers/projects-provider.tsx
```

---

### CASE 7: Layout UI

Example:

```
components/layout/app-navbar.tsx
```

Rule:

* NEVER inside features

---

## End-to-End Flow

---

### /projects

```
URL → /projects
   ↓
app/(dashboard)/projects/page.tsx
   ↓
ProjectsPageClient
   ↓
features/projects/components/
   ↓
uses:
  - project-card
  - modal
  - provider
   ↓
renders UI
```

---

### /projects/[id]

```
URL → /projects/123
   ↓
page.tsx
   ↓
ProjectDetailsClient
   ↓
feature logic
```

---

## Design System (Tailwind v4 + shadcn)

---

### Tokens (global.css)

Defined using `@theme`:

```
--color-primary
--color-foreground
--radius-xl
--shadow-card
```

Rules:

* DO NOT hardcode colors
* ALWAYS use tokens

---

### Styling Rules

* Tailwind utilities in JSX
* NO global component CSS
* NO SCSS architecture

---

### Typography

* Headings → Manrope
* Body → Inter

---

### Components

* Built using shadcn primitives
* Wrapped in `/components/ui`
* Styled via Tailwind

---

## Icons Strategy

---

### Use Icon Library (Default)

Use libraries like:

* lucide-react

For:

* buttons
* navigation
* actions

---

### Use Custom SVG (Selective)

Use Figma SVG ONLY when:

* brand-specific
* not available in library
* visually critical

---

### Rule

> Consistency > pixel-perfect Figma match

---

## Key Principles

* Keep `app/` thin
* Move logic into `features/`
* Keep UI primitives reusable
* Centralize tokens (not styles)
* Avoid global CSS patterns
* Localize ALL user-facing text
* Maintain visual consistency

---

## Notes

* Do NOT mix old “modules/service” architecture here
* This is frontend-first architecture
* Scales by feature, not by file type
* Prefer clarity over abstraction
* Avoid premature optimization

---

[⬆ Back to Table of Contents](#-table-of-contents)
