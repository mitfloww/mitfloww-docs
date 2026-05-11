# Backend Architecture

## 📚 Table of Contents

### Database Access Architecture

* [**Overview**](#overview)
    * [Approaches](#approaches)

        * [Raw SQL](#raw-sql)
        * [ORM using Prisma](#orm-using-prisma)
        * [Drizzle ORM](#drizzle-orm)
    * [Recommended Approach (Hybrid)](#recommended-approach-hybrid)
    * [Target Architecture Principle](#target-architecture-principle)
    * [Separation of Responsibilities](#separation-of-responsibilities)
    * [Hybrid DB Strategy](#hybrid-db-strategy)

        * [Use Drizzle ORM for](#use-drizzle-orm-for)
        * [Use Raw SQL for](#use-raw-sql-for)
    * [Optional: Raw SQL via Drizzle](#optional-raw-sql-via-drizzle)
    * [Future Migration to Node Backend](#future-migration-to-node-backend)
    * [Migration Mapping](#migration-mapping)
    * [Key Takeaways](#key-takeaways)
* [**Problem**](#problem)
* [**Solution**](#solution)
* [**Example**](#example)
    * [DB Layer](#db-layer-libdb)

        * [DB Client with Drizzle](#db-client-with-drizzle)
    * [Schema Layer](#schema-layer-libdbschema)
    * [Repository Layer](#repository-layer-hybrid-drizzle-orm--raw-sql)

    * [Example: file.repository.ts](#example-modulesfilesfilerepositoryts)
    * [Important Rules](#important-rules)
    * [Service Layer](#service-layer-business-logic-only)
    * [API Layer](#api-layer-thin-controllers)
* [**Notes**](#notes)

---

# Database Access Architecture

## Overview

Next.js supports different approaches for database interaction:

### Raw SQL

* **Pros:**

  * Full control
  * Familiar (especially if coming from PostgreSQL / SQL Server)
  * Best for complex queries
* **Cons:**

  * Manual mapping
  * No type safety
  * More boilerplate

---

### ORM using Prisma

* **Pros:**

  * Type-safe
  * Clean queries
  * Schema-driven (similar to EF Core)
* **Cons:**

  * Less flexible for complex queries
  * Harder to optimize joins manually
  * Hides SQL → less control
  * Potential performance overhead

---

### Drizzle ORM

* **Pros:**

  * SQL-like syntax (closer to raw SQL mindset)
  * Type-safe
  * Great for complex queries
  * Easy to mix raw SQL
  * Lightweight and performant
* **Cons:**

  * Slightly more verbose
  * Requires writing schema manually

---

## Recommended Approach (Hybrid)

Use a **hybrid approach**:

* **Drizzle ORM → for CRUD operations**
* **Raw SQL → for complex queries**

This gives:

* Performance
* Flexibility
* Type safety
* Full control when needed

---

## Target Architecture Principle

```
Controllers → Services → Repositories → DB
```

* **Controllers (API routes)** → handle HTTP
* **Services** → business logic (**NO SQL here**)
* **Repositories** → all database access (ORM + SQL)

This ensures:

* Clean separation
* Maintainability
* Easy migration to a dedicated Node.js backend

---

## Separation of Responsibilities

* **API Layer** → transport (HTTP)
* **Service Layer** → business logic
* **Repository Layer** → data access

---

## Hybrid DB Strategy

### Use Drizzle ORM for:

* Inserts
* Updates
* Deletes
* Simple selects

---

### Use Raw SQL for:

* Joins (files + payments + escrow)
* Aggregations (COUNT, SUM)
* Complex filtering/sorting
* Performance-critical queries

---

## Optional: Raw SQL via Drizzle

```ts
import { sql } from 'drizzle-orm';

await db.execute(sql`SELECT * FROM files`);
```

> Note: Prefer `pool.query` for clarity in complex queries.

---

## Future Migration to Node Backend

To migrate to a dedicated backend:

Move:

```
/modules
/lib/db
```

To:

```
Node (Express / Fastify)
```

Replace only:

```
/app/api → Controllers
```

---

## Migration Mapping

| Next.js    | Node Backend |
| ---------- | ------------ |
| route.ts   | controller   |
| service    | service      |
| repository | repository   |
| db client  | db client    |

---

## Key Takeaways

* Use **Drizzle for simplicity + type safety**
* Use **Raw SQL for power + performance**
* Keep **strict separation of layers**
* Design for **failure handling + scalability**
* Ensure **easy migration path from day one**

---

## Problem

---

## Solution

---

## Example

## DB Layer (`/lib/db`)

### DB Client with Drizzle

`/lib/db/client.ts`

```ts
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// Drizzle ORM instance
export const db = drizzle(pool);

// Raw SQL access (for complex queries)
export { pool };
```

---

## Schema Layer (`/lib/db/schema`)

Defines database tables in a **type-safe, code-first manner**.

`/lib/db/schema.ts`

```ts
import { pgTable, uuid, text, integer, timestamp } from 'drizzle-orm/pg-core';

export const files = pgTable('files', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').notNull(),
  url: text('url').notNull(),
  size: integer('size').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
});
```

---

## Repository Layer (Hybrid: Drizzle ORM + Raw SQL)

This layer is responsible for **all database interactions**.

* Drizzle ORM → simple operations
* Raw SQL → complex queries

---

### Example: `/modules/files/file.repository.ts`

```ts
import { db, pool } from '@/lib/db/client';
import { files } from '@/lib/db/schema';

/**
 * 🔹 SIMPLE CRUD (Drizzle ORM)
 */
export async function createFile(data: {
  userId: string;
  url: string;
  size: number;
}) {
  const result = await db
    .insert(files)
    .values({
      userId: data.userId,
      url: data.url,
      size: data.size,
    })
    .returning();

  return result[0];
}

/**
 * 🔹 COMPLEX QUERY (Raw SQL)
 */
export async function getUserFilesWithStats(userId: string) {
  const res = await pool.query(
    `
    SELECT 
      f.*,
      COUNT(p.id) as payment_count,
      COALESCE(SUM(p.amount), 0) as total_earned
    FROM files f
    LEFT JOIN payments p ON p.file_id = f.id
    WHERE f.user_id = $1
    GROUP BY f.id
    ORDER BY f.created_at DESC
    `,
    [userId]
  );

  return res.rows;
}
```

---

## Important Rules

* ❌ NO SQL in Services
* ❌ NO SQL in API routes
* ✅ ALL DB logic in Repositories

---

## Service Layer (Business Logic Only)

`/modules/files/file.service.ts`

```ts
import * as fileRepo from './file.repository';
import { uploadToR2 } from '@/lib/storage/r2';

export async function uploadFile(input: {
  userId: string;
  file: Buffer;
}) {
  // 1. Upload to storage
  const url = await uploadToR2(input.file);

  // 2. Save metadata
  const file = await fileRepo.createFile({
    userId: input.userId,
    url,
    size: input.file.length,
  });

  // 3. Trigger async processing (queue)
  // watermark / compression job

  return file;
}
```

---

## API Layer (Thin Controllers)

`/app/api/files/upload/route.ts`

```ts
import { uploadFile } from '@/modules/files/file.service';

export async function POST(req: Request) {
  const body = await req.json();

  const result = await uploadFile(body);

  return Response.json(result);
}
```

---

## Notes

* Avoid mixing ORM and SQL randomly
* Keep repository layer clean and consistent
* Don’t overuse ORM for complex queries
* Prioritize clarity over abstraction
* `CREATE SCHEMA mitfloww;` to create custom schema
* Run drizzle migration using:
    * `npx drizzle-kit generate`
    * `npx drizzle-kit push`
* For cloud DB:
    * ```DATABASE_URL=<neon_url> npx drizzle-kit migrate```
* To ensure schema before running migrations:
    * ```npm run db:bootstrap```
---

[⬆ Back to Table of Contents](#-table-of-contents)