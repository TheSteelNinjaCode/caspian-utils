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

High-priority rule: when `caspian.config.json` has `prisma: true`, Python-side database access must use the generated Prisma Python ORM. Route `page()` functions, route-owned `@rpc()` actions, auth handlers, upload flows, and shared helpers should import the generated client from `src/lib/prisma/**` instead of inventing direct driver calls, hand-written SQL wrappers, JSON manifests as active stores, app-specific HTTP fetches, or browser-side database fetches. For normal reads and writes, default to Prisma methods such as `find_many`, `find_unique`, `create`, `update`, `delete`, `delete_many`, aggregates, includes, and transactions. Use raw SQL only through Prisma as a narrow fallback when the generated ORM cannot express a query clearly, and treat that fallback as provider-specific code that may break when a project moves between SQLite, MySQL, and PostgreSQL.

Treat `caspian.config.json` as the single source of truth for whether Prisma is enabled in a workspace. If `prisma` is false and the user wants Prisma, ask first, then update `caspian.config.json` and run `npx casp update project` before assuming Prisma-managed files exist.

## Overview

The standard Prisma flow in Caspian is:

1. Define models in `prisma/schema.prisma`.
2. Configure `DATABASE_URL` in `.env`.
3. Run `npx prisma migrate dev` after schema changes so the development database and migration history stay aligned.
4. If the change requires seed data, run `npx prisma generate`, then request explicit user approval before running `npx prisma db seed`.
5. Run `npx ppy generate` so the Python ORM classes stay aligned with the updated schema.
6. Reuse the shared Python database layer in `src/lib/prisma/` when Python route or RPC code needs database access, and never hand-edit generated ORM classes.

Use this workflow instead of writing raw SQL first. For normal CRUD and relation work, stay on the generated Prisma API. Drop to raw SQL only through Prisma when a query cannot be expressed clearly with the generated client, and assume that raw SQL requires extra review for cross-database portability.

## Seed Command Safety

Treat `npx prisma db seed` as a destructive data operation, not as a routine regeneration command. A project's `prisma/seed.ts` may delete existing rows, reset lookup tables, replace users, or otherwise rewrite the current datasource before inserting seed data. If `DATABASE_URL` points at a shared, staging, or production database, running the seed command can delete production information.

Before any AI agent runs `npx prisma db seed`, it must:

1. Read `prisma.config.ts` and `prisma/seed.ts` to understand what the seed command will execute.
2. State the exact command it intends to run: `npx prisma db seed`.
3. Warn the user clearly that the command can delete or overwrite database data.
4. Confirm the active datasource or `DATABASE_URL` when practical, without exposing secrets.
5. Wait for explicit user approval before executing the command.

Do not run `npx prisma db seed` merely because it appears in the standard workflow. Ask first, and only continue after the user has approved that specific seed operation.

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

For most local development, SQLite is the simplest starting point. For production or higher concurrency workloads, prefer PostgreSQL or MySQL. Regardless of the starting provider, keep application queries on the Prisma ORM whenever possible so schema and query behavior stay easier to migrate across providers.

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
2. If seed data or the seed script needs the new schema, run `npx prisma generate`, then request explicit user approval before running `npx prisma db seed`.
3. Run `npx ppy generate` to refresh the generated Python ORM classes.

Do not hand-edit generated Prisma or Python ORM output. Treat `prisma/schema.prisma` as the source of truth and regenerate from it.

## Command Reference

Use these commands for the normal Prisma lifecycle in Caspian:

- `npx prisma migrate dev` creates and applies a development migration. This is the first command to run after changing `prisma/schema.prisma`.
- `npx prisma generate` compiles `schema.prisma` into the generated Node Prisma client. Run it before `npx prisma db seed` when the updated seed flow depends on the new schema.
- `npx prisma db seed` runs the configured seeding script. This may delete or overwrite existing database data; an AI agent must warn the user and receive explicit approval before executing it.
- `npx ppy generate` regenerates the Python ORM classes used by the app. Use this after every schema change.
- `npx prisma migrate deploy` applies pending migrations in deployment environments.
- `npx prisma db push` syncs schema changes without creating a migration file, which is useful for prototyping.
- `npx prisma studio` opens the Prisma data browser.

Default rule:

- When `prisma/schema.prisma` changes, use this order: `npx prisma migrate dev`; if seeding is involved, run `npx prisma generate`, ask for explicit approval before `npx prisma db seed`, then run `npx ppy generate`.
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

If the project ships an app-owned Python Prisma-style layer under `src/lib/prisma/`, treat it as async-first and reuse it from route and RPC code. In a Prisma-enabled project, this is the required data-access path for Python code.

Example:

```python
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

Keep route-specific Prisma I/O in that route's `page()` or `@rpc()` actions in `src/app/**/index.py`. The installed layout engine supports synchronous and async `layout()` results, but layout work should stay focused on shared subtree props or metadata. Move Prisma helpers into `src/lib/**` only when the same behavior is reused by multiple routes, features, or integrations.

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

Null filtering follows Prisma semantics: `{"name": None}` or `{"name": {"equals": None}}` compiles to `IS NULL`, and `{"name": {"not": None}}` to `IS NOT NULL`. String filters `contains`, `startsWith`, and `endsWith` match the user's value literally (`%` and `_` are escaped, not treated as wildcards). `mode: "insensitive"` is PostgreSQL-only; MySQL is usually case-insensitive already through its collation, and SQLite needs `COLLATE NOCASE` on the column.

Pagination uses `take` and `skip` for offset windows, or `cursor` anchored at a unique row:

```python
page = await prisma.post.find_many(
    order_by={"id": "asc"},
    cursor={"id": last_seen_id},
    skip=1,   # skip the cursor row itself
    take=20,
)
```

Always pass an `order_by` with `cursor`: with a plain scalar ordering the client pushes the cursor window into SQL and fetches only the requested page; without one it must scan the matching rows to locate the cursor position. A negative `take` returns the rows before the cursor. `distinct` and `select`/`omit` projections compose with all of the above.

### Update

```python
updated_user = await prisma.user.update(
    where={"id": "123"},
    data={
        "name": "New Name",
    },
)
```

Numeric fields support the atomic update operators `set`, `increment`, `decrement`, `multiply`, and `divide` — exactly one operator per field. The arithmetic runs inside the SQL `UPDATE` itself (`SET views = views + ?`), so concurrent updates cannot lose increments. Matching Prisma semantics: a `NULL` column stays `NULL`, `divide` on `Int` fields truncates, and dividing by zero raises a validation error.

```python
await prisma.post.update(
    where={"id": "123"},
    data={"views": {"increment": 1}},
)
```

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

## Generated Client Surface

The generated delegates expose the standard Prisma method set: `find_many`, `find_unique`, `find_first`, `find_unique_or_throw`, `find_first_or_throw`, `create`, `create_many`, `create_many_and_return`, `update`, `update_many`, `update_many_and_return`, `upsert`, `delete`, `delete_many`, `count`, `aggregate`, and `group_by`. Reads accept `where`, `select`, `omit`, `include`, `order_by`, `take`, `skip`, `cursor`, and `distinct`. Relation filters (`some`, `none`, `every`, `is`, `isNot`) work inside `where`, and write payloads accept nested relation operations (`create`, `connect`, `connectOrCreate`, `set`, `disconnect`, `update`, `updateMany`, `upsert`, `delete`, `deleteMany`).

Relation loading is batched: including a relation for N parent rows issues one query per relation per nesting level, not one query per parent, so `include` does not create N+1 query patterns. Filters a provider cannot express in SQL fall back to a slower Python evaluation path with a logged warning instead of failing.

## Error Handling

The generated client raises Prisma-style errors, importable from the same package:

- `RecordNotFoundError` (Prisma code `P2025`) when `update`, `delete`, `upsert`-related lookups, `find_unique_or_throw`, or `find_first_or_throw` target a record that does not exist.
- `PrismaClientValidationError` for invalid query shapes, such as combining `select` with `include`, unsupported `order_by` shapes, or malformed atomic-update payloads.

```python
from src.lib.prisma import prisma, RecordNotFoundError

try:
    await prisma.user.update(where={"id": user_id}, data={"name": name})
except RecordNotFoundError:
    ...  # surface a 404-style response instead of a 500
```

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

Transactions accept Prisma-style options — `isolation_level` (validated against what the provider supports), plus `max_wait` and `timeout` in milliseconds. `prisma.batch([...])` runs a list of operations inside a single transaction.

### Raw SQL Fallback

Use this escape hatch sparingly. Raw SQL is not the default query style in a Prisma-enabled Caspian project.

Before using `query_raw()` or `execute_raw()`, confirm that the generated Prisma API cannot express the query clearly with normal methods such as `find_many`, `find_unique`, `create`, `update`, `delete`, `delete_many`, `aggregate`, relation `include`, or transactions.

Important portability warning:

- Raw SQL is provider-specific. Placeholder style, identifier quoting, JSON operators, case-sensitivity rules, date functions, and aggregate behavior can differ across SQLite, MySQL, and PostgreSQL.
- A raw query that works in SQLite may fail after a move to PostgreSQL or MySQL even when the Prisma schema migrates cleanly.
- If a project may switch providers, prefer Prisma ORM methods first and treat raw SQL as an exception that needs provider-aware review.

```python
users = await prisma.user.find_many(
    where={
        "email": {"contains": "@gmail.com"},
    },
)
```

If raw SQL is still unavoidable, keep it tightly scoped, document why the ORM was insufficient, and verify it against the current datasource provider in `prisma/schema.prisma`.

## Provider Portability And Switching

The generated Python client is compiled for one datasource provider at a time; the SQL dialect is baked in at generation. Switching a project between SQLite, PostgreSQL, and MySQL is supported and designed to be smooth, but it is a regeneration, not a config flip:

1. Change `provider` in the `datasource` block of `prisma/schema.prisma`.
2. Point `DATABASE_URL` in `.env` at the new database.
3. Recreate the database structure. Migration SQL under `prisma/migrations/` is dialect-specific and does not port across providers; re-baseline the migrations or use `npx prisma db push` against the new datasource.
4. Run `npx ppy generate` to regenerate the Python client for the new provider.

Guard rails to rely on:

- The client fails fast at connect time when `DATABASE_URL` points at a different provider than the one it was generated for, with an error message that names the fix (regenerate).
- On the MySQL family the client probes the live server once and adapts to MySQL versus MariaDB and to the server version automatically (RETURNING support, JSON functions, window functions).
- Filters a provider cannot express in SQL degrade to Python evaluation with a logged warning instead of failing, so a switch tends to change performance before it changes behavior.

Keep the schema inside the portable subset when a project may switch providers:

- Scalar list fields (`String[]` and friends) are PostgreSQL-only; model list-like data as `Json` instead.
- Enums are not supported by Prisma on SQLite.
- Avoid `@db.*` native type attributes unless the project is committed to one provider.

API differences that follow the real database engines (these match Prisma TypeScript behavior):

- `mode: "insensitive"` string filtering is PostgreSQL-only.
- `create_many_and_return` is unavailable on MySQL, and its `skip_duplicates` option is unavailable on SQLite.
- String comparison case sensitivity differs: `equals` is case-insensitive under default MySQL collations; `contains`/`startsWith`/`endsWith` are case-insensitive on SQLite and MySQL but case-sensitive on PostgreSQL.
- `update_many` reports matched rows on PostgreSQL and SQLite but changed rows on MySQL.

After a switch, run the project's database-backed tests against the new provider before treating the migration as done.

## Recommended Project Rules

- Keep the schema in `prisma/schema.prisma` and follow the required regeneration order after changes: `npx prisma migrate dev`, optional seed flow, then `npx ppy generate`.
- Reuse `src/lib/prisma/` for Python-side database access instead of creating a second bridge.
- When Prisma is enabled, do not bypass the Prisma Python ORM with direct database drivers, custom fetch functions, JSON files as active stores, or browser-side database access.
- Never hand-edit generated Prisma or Python ORM classes.
- Keep reusable database helpers in `src/lib/`, and keep route-specific orchestration in `src/app/`.
- Use `await` with Prisma operations.
- Convert Prisma objects to template-safe dictionaries when rendering HTML.
- Validate incoming mutation data before calling `create`, `update`, or `delete` operations.
- Prefer Prisma queries over raw SQL. If raw SQL is unavoidable, keep it provider-aware, narrowly scoped, and documented so future database-provider migrations do not silently break application behavior.

## AI Retrieval Notes

If an AI agent is working on a Caspian app with Prisma enabled, apply these rules first.

- Treat Prisma as the default ORM and persistence layer.
- Read `prisma/schema.prisma` before proposing model, relation, or field changes.
- When `prisma/schema.prisma` changes, run `npx prisma migrate dev` first.
- If the schema change requires seed data, run `npx prisma generate`, then request explicit user approval before running `npx prisma db seed`.
- Run `npx ppy generate` after schema changes to refresh the Python ORM classes.
- Never hand-edit generated Prisma or Python ORM classes.
- Read `prisma.config.ts` and `prisma/seed.ts` when you need the current project's Prisma tooling examples.
- Before changing the datasource provider, read the Provider Portability And Switching section above: a provider switch requires regeneration plus provider-aware review of the schema and any provider-gated query features.
- Reuse the existing `src/lib/prisma/` package when the Python app needs database access.
- Treat use of the Prisma Python ORM as mandatory for Python-side database reads and writes in Prisma-enabled projects.
- For file managers and uploads, persist metadata in Prisma and keep blob storage separate. See [file-uploads.md](./file-uploads.md).
- Put reusable database helpers in `src/lib/`; keep route and RPC orchestration in `src/app/`.
- Use `async def page()` for route-specific first-render reads. Use `layout()` only for shared subtree props or metadata, and use `@rpc()` plus `pp.rpc()` for browser-triggered reads and writes.
- Check `fetch-data.md` for route versus RPC guidance and `validation.md` before writing public mutations.
