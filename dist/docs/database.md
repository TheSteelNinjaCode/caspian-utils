---
title: Database
description: Use Prisma in Caspian with the generated async Python client, schema-first workflow, and the shared imports under `src/lib/prisma/`.
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

This page documents the default Caspian database workflow with Prisma ORM.

Treat Prisma as the default persistence layer when a Caspian app includes database support. The Prisma schema is the source of truth, the generated Python client is the runtime interface, and route code should import the shared client from `src.lib.prisma` instead of constructing ad hoc database connections inside each page or RPC action.

## Overview

The standard Prisma flow in Caspian is:

1. Define models in `prisma/schema.prisma`.
2. Configure `DATABASE_URL` in `.env`.
3. Run `npx ppy generate` after schema changes.
4. Import the generated async client from `src.lib.prisma.db` or `src.lib.prisma`.
5. Use Prisma calls inside `page()`, `layout()`, or `@rpc()` actions.

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

Model your data in `prisma/schema.prisma`. This file is the source of truth for both the database structure and the generated Python client API.

Example:

```prisma
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-py"
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

After changing the schema, regenerate the Python client before using new models or fields in app code.

## Command Reference

Use these commands for the normal Prisma lifecycle in Caspian:

- `npx ppy generate` compiles `schema.prisma` into the generated Python client. Run this after every schema change.
- `npx prisma migrate dev` creates and applies a development migration.
- `npx prisma migrate deploy` applies pending migrations in deployment environments.
- `npx prisma db push` syncs schema changes without creating a migration file, which is useful for prototyping.
- `npx prisma db seed` runs the configured seeding script.
- `npx prisma studio` opens the Prisma data browser.

Default rule:

- Use `npx ppy generate` every time the schema changes.
- Use migrations for tracked application changes.
- Use `db push` only when you intentionally want a faster, migration-free prototype loop.

## Generated Python Client Files

In a Prisma-enabled Caspian app, the generated and shared Python imports live under `src/lib/prisma/`.

### `src/lib/prisma/db.py`

This module exposes the shared Prisma client instance used by route code and RPC actions.

Typical usage:

```python
from src.lib.prisma.db import prisma

users = await prisma.user.find_many()
```

Use this import as the default way to access the database. Do not instantiate new Prisma clients inside each request handler unless you are solving a very specific lifecycle problem.

### `src/lib/prisma/models.py`

This module contains the generated Python model types derived from `prisma/schema.prisma`.

Use it when you need explicit model imports for typing, serialization, or helper code that works with generated record objects.

Typical usage pattern:

```python
from src.lib.prisma import models

def as_public_user(user: models.User) -> dict:
    return {
        "id": user.id,
        "email": user.email,
        "name": user.name,
    }
```

Treat these classes as generated artifacts. Change the schema, then regenerate, instead of editing model definitions by hand.

### `src/lib/prisma/__init__.py`

This module is the package entry point for Prisma imports.

If it re-exports the shared client and generated model helpers, application code can use a shorter import shape such as:

```python
from src.lib.prisma import prisma, models
```

Prefer this package-level import when it keeps route code clearer. Prefer direct imports from `db.py` when you want to make the client source explicit.

## Async Client Usage

The Prisma client in Caspian should be treated as async-first. Use `await` for database operations.

Example:

```python
from src.lib.prisma.db import prisma

async def page():
    users = await prisma.user.find_many()

    return {
        "users": [user.to_dict() for user in users],
    }
```

Prisma calls fit naturally in:

- `async def page()` for first-render data
- `async def layout()` for shared section data
- `@rpc()` actions for browser-triggered reads and writes

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
async with prisma.tx() as tx:
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

- Keep the schema in `prisma/schema.prisma` and regenerate the client after changes.
- Import the shared client from `src.lib.prisma.db` or `src.lib.prisma`, not from ad hoc connection code scattered across routes.
- Keep reusable database helpers in `src/lib/`, and keep route-specific orchestration in `src/app/`.
- Use `await` with Prisma operations.
- Convert Prisma objects to template-safe dictionaries when rendering HTML.
- Validate incoming mutation data before calling `create`, `update`, or `delete` operations.
- Prefer Prisma queries over raw SQL, and prefer raw SQL over undocumented custom query helpers.

## AI Routing Notes

If an AI agent is working on a Caspian app with Prisma enabled, apply these rules first.

- Treat Prisma as the default ORM and persistence layer.
- Read `prisma/schema.prisma` before proposing model, relation, or field changes.
- Run `npx ppy generate` after schema changes so the Python client matches the schema.
- Import the shared client from `src.lib.prisma.db` or the `src.lib.prisma` package entry point.
- Treat `src/lib/prisma/models.py` as generated model output, not hand-authored business logic.
- Put reusable database helpers in `src/lib/`; keep route and RPC orchestration in `src/app/`.
- Use `page()` or `layout()` for first-render reads and `@rpc()` plus `pp.rpc()` for browser-triggered reads and writes.
- Check `fetch-data.md` for route versus RPC guidance and `validation.md` before writing public mutations.
