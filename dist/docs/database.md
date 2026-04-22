---
title: Database
description: Use this page when the task mentions Prisma, `schema.prisma`, `prisma.config.ts`, migrations, `seed.ts`, Prisma Client, or `ppy generate` in a Caspian project.
related:
  title: Related docs
  description: Use the project structure guide to place database files correctly, then use the fetch-data guide when Prisma calls need to power route rendering or RPC actions.
  links:
    - /docs/commands
    - /docs/project-structure
    - /docs/fetch-data
    - /docs/validation
    - /docs/index
---

This page documents the Prisma workflow for Caspian projects where `caspian.config.json` enables Prisma.

When a project enables Prisma, use `prisma/schema.prisma` for schema management, `prisma.config.ts` for Prisma config and seed wiring, and reuse an app-owned `src/lib/prisma/` package when the project includes one.

Treat `caspian.config.json` as the single source of truth for whether Prisma is enabled in a workspace. If `prisma` is false and the user wants Prisma, ask first, then update `caspian.config.json` and run `npx casp update project` before assuming Prisma-managed files exist.

## Overview

The standard Prisma flow in Caspian is:

1. Define models in `prisma/schema.prisma`.
2. Configure `DATABASE_URL` in `.env`.
3. Run `npx prisma migrate dev` after schema changes so the development database and migration history stay aligned.
4. If the change requires seed data, run `npx prisma generate` and then `npx prisma db seed`.
5. Run `npx ppy generate` so the Python ORM classes stay aligned with the updated schema.
6. Reuse the shared Python database layer in `src/lib/prisma/` when Python route or RPC code needs database access, and never hand-edit generated ORM classes.

Use this workflow instead of writing raw SQL first. Drop to raw SQL only when a query cannot be expressed clearly with the generated client.

## Environment Setup

Add your database connection string to `.env`.

Example for SQLite in development:

```env
DATABASE_URL="file:./prisma/dev.db"
```

Example for PostgreSQL in async-friendly production environments:

```env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
```

For most local development, SQLite is the simplest starting point. For production or higher concurrency workloads, prefer PostgreSQL or MySQL.

## Global Prisma Configuration

Use `prisma.config.ts` for seed configuration and ORM-level behavior.

Example:

```ts
export default {
  seed: {
    import: "./prisma/seed.ts",
    autoRun: true,
  },
  client: {
    logLevel: "info",
  },
}
```

Keep schema in `prisma/schema.prisma`, connection secrets in `.env`, and generation or seed behavior in `prisma.config.ts`.

## Define Your Schema

Model your data in `prisma/schema.prisma`. This file is the source of truth for the database structure used by both the generated Node Prisma client and the app-owned Python layer in `src/lib/prisma/`.

Example:

```prisma
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
    provider = "prisma-client-js"
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
}

model Post {
  id        String   @id @default(cuid())
  title     String
  published Boolean  @default(false)
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
}
```

After changing the schema, use this order:

1. Run `npx prisma migrate dev`.
2. If seed data or the seed script needs the new schema, run `npx prisma generate` and then `npx prisma db seed`.
3. Run `npx ppy generate` to refresh the generated Python ORM classes.

Do not hand-edit generated Prisma or Python ORM output. Treat `prisma/schema.prisma` as the source of truth and regenerate from it.

## Command Reference

Use these commands for the normal Prisma lifecycle in Caspian:

- `npx prisma migrate dev` creates and applies a development migration. This is the first command to run after changing `prisma/schema.prisma`.
- `npx prisma generate` compiles `schema.prisma` into the generated Node Prisma client. Run it before `npx prisma db seed` when the updated seed flow depends on the new schema.
- `npx prisma db seed` runs the configured seeding script.
- `npx ppy generate` regenerates the Python ORM classes used by the app. Use this after every schema change.
- `npx prisma migrate deploy` applies pending migrations in deployment environments.
- `npx prisma db push` syncs schema changes without creating a migration file, which is useful for prototyping.
- `npx prisma studio` opens the Prisma data browser.

Default rule:

- When `prisma/schema.prisma` changes, use this order: `npx prisma migrate dev`; if seeding is involved, run `npx prisma generate` and `npx prisma db seed`; then run `npx ppy generate`.
- Never hand-edit generated Prisma or Python ORM classes, and never replace `npx ppy generate` with a manual class update.
- Use migrations for tracked application changes.
- Use `db push` only when you intentionally want a faster, migration-free prototype loop.

## Prisma Files To Inspect

In a Prisma-enabled Caspian project, inspect these locations:

### `prisma/schema.prisma`

This file is the schema source of truth. Confirm the actual datasource and generator configuration in the current project.

### `prisma.config.ts`

This file configures the schema path, migrations path, and the seed entry point. Confirm the actual seed command and import path in the current project.

### `prisma/seed.ts`

This file is the project-local example of code that imports and uses the generated Prisma client.

### `node_modules/@prisma/client/`

After `npx prisma generate`, Prisma writes the generated JavaScript client into the normal `@prisma/client` package location.

### `src/lib/prisma/`

If the project includes an app-owned async Python database package here, it typically exports `prisma`, `PrismaClient`, generated models, and helper types.

If Python route or RPC code needs database access, import from `src.lib.prisma` and keep that import path explicit in application code. Refresh generated classes with `npx ppy generate` instead of editing them manually.

## Python Route Usage

If the project ships an app-owned Python Prisma-style layer under `src/lib/prisma/`, treat it as async-first and reuse it from route and RPC code.

Example:

```python
from casp.layout import render_page
from src.lib.prisma import prisma

async def page():
    users = await prisma.user.find_many()

    return {
        "users": [user.to_dict() for user in users],
    }
```

Prisma calls fit naturally in:

- `async def page()` for first-render data
- `@rpc()` actions for browser-triggered reads and writes

Keep Prisma I/O out of `layout.py` in the current runtime because `casp.layout` calls `layout()` synchronously.

See `fetch-data.md` for the recommended route-render versus RPC split.

## CRUD Examples

### Create

```python
user = await prisma.user.create(
    data={
        "email": "alice@example.com",
        "name": "Alice",
    }
)
```

### Read

Filtering and sorting:

```python
posts = await prisma.post.find_many(
    where={
        "published": True,
        "title": {"contains": "Caspian"},
    },
    order_by={
        "title": "asc",
    },
)
```

Selection and relations:

```python
user = await prisma.user.find_unique(
    where={"email": "alice@example.com"},
    include={"posts": True},
)
```

### Update

```python
updated_user = await prisma.user.update(
    where={"id": "123"},
    data={
        "name": "New Name",
    },
)
```

If your schema includes numeric counters, Prisma also supports atomic update operators such as `increment` and `decrement`.

### Delete

```python
await prisma.user.delete(where={"id": "123"})

await prisma.post.delete_many(
    where={
        "published": False,
    }
)
```

## Using Prisma In RPC Actions

Prisma is a strong fit for Caspian RPC because both are async-friendly.

Example:

```python
from casp.rpc import rpc
from casp.validate import Validate
from src.lib.prisma import prisma

@rpc()
async def create_user(email: str, name: str | None = None):
    if Validate.with_rules(email, "required|email") is not True:
        raise ValueError("A valid email address is required")

    user = await prisma.user.create(
        data={
            "email": email,
            "name": name,
        }
    )

    return user.to_dict()
```

Use validation before writes, especially for form payloads and public RPC actions. See `validation.md` for the recommended boundary checks.

## Upload Metadata Pattern

When a Caspian file manager needs durable metadata, keep the blob and the metadata in separate layers.

- Write the uploaded blob to disk or object storage.
- Persist the durable metadata in Prisma.
- Store fields such as owner relation, original name, stored name, asset path, MIME type, collection, kind, size, and uploaded timestamp in the Prisma model.
- Remove both the stored blob and the Prisma row during delete flows.
- Do not use JSON manifests as the primary metadata store when Prisma is enabled.

See [file-uploads.md](./file-uploads.md) for the route-local RPC plus public asset storage pattern.

## Advanced Features

### Aggregations

```python
stats = await prisma.post.aggregate(
    _count={"_all": True},
    where={"published": True},
)

print(stats["_count"]["_all"])
```

### Transactions

```python
async with prisma.transaction() as tx:
    user = await tx.user.create(
        data={
            "email": "alice@example.com",
            "name": "Alice",
        }
    )
    await tx.post.create(
        data={
            "title": "Welcome post",
            "authorId": user.id,
        }
    )
```

### Raw SQL Fallback

```python
users = await prisma.query_raw(
    "SELECT * FROM User WHERE email LIKE ?",
    "%@gmail.com",
)
```

Use raw SQL sparingly. Prefer the generated Prisma API when the query can be expressed clearly there.

## Recommended Project Rules

- Keep the schema in `prisma/schema.prisma` and follow the required regeneration order after changes: `npx prisma migrate dev`, optional seed flow, then `npx ppy generate`.
- Reuse `src/lib/prisma/` for Python-side database access instead of creating a second bridge.
- Never hand-edit generated Prisma or Python ORM classes.
- Keep reusable database helpers in `src/lib/`, and keep route-specific orchestration in `src/app/`.
- Use `await` with Prisma operations.
- Convert Prisma objects to template-safe dictionaries when rendering HTML.
- Validate incoming mutation data before calling `create`, `update`, or `delete` operations.
- Prefer Prisma queries over raw SQL, and prefer raw SQL over undocumented custom query helpers.

## AI Retrieval Notes

If an AI agent is working on a Caspian app with Prisma enabled, apply these rules first.

- Treat Prisma as the default ORM and persistence layer.
- Read `prisma/schema.prisma` before proposing model, relation, or field changes.
- When `prisma/schema.prisma` changes, run `npx prisma migrate dev` first.
- If the schema change requires seed data, run `npx prisma generate` and then `npx prisma db seed`.
- Run `npx ppy generate` after schema changes to refresh the Python ORM classes.
- Never hand-edit generated Prisma or Python ORM classes.
- Read `prisma.config.ts` and `prisma/seed.ts` when you need the current project's Prisma tooling examples.
- Reuse the existing `src/lib/prisma/` package when the Python app needs database access.
- For file managers and uploads, persist metadata in Prisma and keep blob storage separate. See [file-uploads.md](./file-uploads.md).
- Put reusable database helpers in `src/lib/`; keep route and RPC orchestration in `src/app/`.
- Use `async def page()` for first-render reads. Keep `layout()` synchronous in the current runtime, and use `@rpc()` plus `pp.rpc()` for browser-triggered reads and writes.
- Check `fetch-data.md` for route versus RPC guidance and `validation.md` before writing public mutations.
