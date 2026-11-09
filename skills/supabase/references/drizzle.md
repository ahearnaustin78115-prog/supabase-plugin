# Connecting to Supabase Postgres with Drizzle

[Drizzle ORM](https://github.com/drizzle-team/drizzle-orm) is a TypeScript ORM for SQL databases with a type-safe query API.

## Install

```shell
npm i drizzle-orm postgres
npm i -D drizzle-kit
```

## Define a schema

```ts
import { pgTable, serial, text, varchar } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  fullName: text("full_name"),
  phone: varchar("phone", { length: 256 }),
});
```

## Connect

Set `DATABASE_URL` to the URI from the project's **Connect** panel. For pooled application connections, use the **Shared pooler** URI and replace its password placeholder. Keep the URL in an environment variable and do not commit credentials.

```ts
import "dotenv/config";

import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";

const databaseUrl = process.env.DATABASE_URL;
if (!databaseUrl) throw new Error("DATABASE_URL is not set");

const connectionUrl = new URL(databaseUrl);

// In a local Supabase Docker network, the project-specific hostname
// supabase_db_<project-ref> resolves through the `db` service name.
if (connectionUrl.hostname.startsWith("supabase_db_")) {
  connectionUrl.hostname = "db";
}

// Disable prepared statements for transaction-mode pooling.
export const client = postgres(connectionUrl.href, { prepare: false });
export const db = drizzle(client);
```

If you use Drizzle instead of the Supabase Data API (PostgREST), you can disable the Data API in [API Settings](https://supabase.com/dashboard/project/_/settings/api).

References:

- [Supabase: Connecting to Postgres](https://supabase.com/docs/guides/database/connecting-to-postgres)
- [Drizzle ORM: PostgreSQL](https://orm.drizzle.team/docs/get-started-postgresql)
