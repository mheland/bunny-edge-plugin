# Bunny Database — Edge-Replicated Database Reference

Bunny Database is a fully managed relational database built on **libSQL** (a fork of SQLite). It provides globally replicated SQL databases with usage-based billing. Currently in **Public Preview**.

## Architecture

### Storage & Compute Separation

Data resides in designated storage locations while compute instances run dynamically across regions. This enables low-latency reads without moving data.

**Storage locations** (data at rest):
- Toronto, Canada (North America)
- Frankfurt, Germany (Europe)

### Region Types

**Primary regions** — handle all write operations. Multiple can be configured, but only one is active at a time. The system auto-selects based on latency. Primary assignment may change when a database reactivates after idle.

**Replication regions** — dynamically provisioned read replicas. Handle read requests locally but proxy write operations to the primary region.

### Replication Model

- **Eventual consistency** — the primary streams changes to replicas in real-time, but replication lag depends on network latency.
- **No read-your-writes guarantee** on replicas. If you need to read immediately after writing, target the primary.
- Stale replicas track their version number and proxy requests to the primary until synced.
- 60-second timeout during primary transitions.

### Durability

- Writes are durable when appended to the primary's WAL (Write-Ahead Log).
- Data persists to storage every **10 seconds** or every **4,096 frames** (page changes), whichever comes first.
- **Maximum data loss window**: up to 10 seconds of uncommitted writes during primary failover.
- Hourly vacuum snapshots while online; full snapshots when databases go idle.
- Recovery: latest snapshot + WAL segment replay.

### Idle Behavior

Inactive databases are automatically archived to object storage and removed from active regions to reduce costs. They reactivate on first request.

---

## Setup & Connection

### Creating a Database

1. Go to Dashboard > Edge Platform > Database
2. Provide a unique database identifier (becomes part of connection URL)
3. Choose region strategy:
   - **Automatic** — system optimizes placement
   - **Single Region** — one location, no replication (good for dev)
   - **Manual** — choose storage and replication regions

### Access Tokens

Generated from Dashboard > Database > [Select Database] > Access > **Generate Tokens**.

Two token types created simultaneously:
- **Full Access** — read + write
- **Read Only** — SELECT queries only

**Important:**
- Tokens are shown only once. If lost, generate new ones.
- **Regenerating tokens invalidates ALL existing tokens across ALL databases.**

### Connection URL Format

```
libsql://[your-database-id].lite.bunnydb.net
```

---

## Connecting from Edge Scripts

The recommended integration path:

1. Go to Database > [Your Database] > Access
2. Click **Generate Tokens**
3. Click **Add Secrets to Edge Script**
4. Select which Edge Script to connect

This injects two environment variables into the script:
- `BUNNY_DATABASE_URL`
- `BUNNY_DATABASE_AUTH_TOKEN`

### Basic Usage

```ts
import { createClient } from "@libsql/client/web";
import process from "node:process";

const client = createClient({
  url: process.env.BUNNY_DATABASE_URL,
  authToken: process.env.BUNNY_DATABASE_AUTH_TOKEN,
});

const result = await client.execute("SELECT * FROM users");
console.log(result.rows);
```

**Important:** Use `@libsql/client/web` (the web import path), not `@libsql/client` directly, when running in Edge Scripts.

### Full CRUD Example

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";
import { createClient } from "@libsql/client/web";
import process from "node:process";

const client = createClient({
  url: process.env.BUNNY_DATABASE_URL,
  authToken: process.env.BUNNY_DATABASE_AUTH_TOKEN,
});

// Initialize schema
await client.execute(`
  CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
  )
`);

BunnySDK.net.http.serve(async (request: Request): Response | Promise<Response> => {
  const url = new URL(request.url);

  // List users
  if (url.pathname === "/users" && request.method === "GET") {
    const result = await client.execute("SELECT * FROM users");
    return new Response(JSON.stringify(result.rows), {
      headers: { "content-type": "application/json" },
    });
  }

  // Create user
  if (url.pathname === "/users" && request.method === "POST") {
    const body = await request.json();
    const result = await client.execute({
      sql: "INSERT INTO users (name, email) VALUES (?, ?)",
      args: [body.name, body.email],
    });
    return new Response(JSON.stringify({ id: Number(result.lastInsertRowid) }), {
      status: 201,
      headers: { "content-type": "application/json" },
    });
  }

  // Get user by ID
  if (url.pathname.startsWith("/users/") && request.method === "GET") {
    const id = url.pathname.split("/")[2];
    const result = await client.execute({
      sql: "SELECT * FROM users WHERE id = ?",
      args: [id],
    });
    if (result.rows.length === 0) {
      return new Response(JSON.stringify({ error: "Not found" }), {
        status: 404,
        headers: { "content-type": "application/json" },
      });
    }
    return new Response(JSON.stringify(result.rows[0]), {
      headers: { "content-type": "application/json" },
    });
  }

  // Delete user
  if (url.pathname.startsWith("/users/") && request.method === "DELETE") {
    const id = url.pathname.split("/")[2];
    await client.execute({
      sql: "DELETE FROM users WHERE id = ?",
      args: [id],
    });
    return new Response(null, { status: 204 });
  }

  return new Response("Not Found", { status: 404 });
});
```

---

## TypeScript SDK Reference

### Installation

```bash
npm install @libsql/client
```

### Creating a Client

```ts
import { createClient } from "@libsql/client/web";

const client = createClient({
  url: process.env.BUNNY_DATABASE_URL,       // libsql://...
  authToken: process.env.BUNNY_DATABASE_AUTH_TOKEN,
});
```

### Simple Queries

```ts
// No parameters
const result = await client.execute("SELECT * FROM users");

// Positional parameters
const result = await client.execute({
  sql: "SELECT * FROM users WHERE id = ?",
  args: [1],
});

// Named parameters (supports :name, @name, $name)
const result = await client.execute({
  sql: "INSERT INTO users VALUES (:name)",
  args: { name: "Kit" },
});
```

### ResultSet

```ts
{
  rows: Row[],              // array of row objects
  columns: string[],        // column names
  rowsAffected: number,     // for INSERT/UPDATE/DELETE
  lastInsertRowid: bigint,  // last auto-increment ID
}
```

### Batch Transactions

Execute multiple statements atomically — auto-rollback on failure:

```ts
const results = await client.batch(
  [
    { sql: "INSERT INTO users VALUES (?)", args: ["Kit"] },
    { sql: "INSERT INTO users VALUES (?)", args: ["Sam"] },
  ],
  "write",
);
```

Modes:
- `"write"` — read/write access
- `"read"` — SELECT only
- `"deferred"` — starts as read, promotes to write on first write

### Interactive Transactions

Manual commit/rollback control. **5-second timeout** — keep transactions short.

```ts
const tx = await client.transaction("write");
try {
  const result = await tx.execute({
    sql: "SELECT balance FROM accounts WHERE id = ?",
    args: [1],
  });

  const balance = result.rows[0].balance as number;
  await tx.execute({
    sql: "UPDATE accounts SET balance = ? WHERE id = ?",
    args: [balance - 100, 1],
  });

  await tx.commit();
} catch (e) {
  await tx.rollback();
}
```

Transaction methods: `execute()`, `commit()`, `rollback()`, `close()`.

---

## SQL API (HTTP)

For direct HTTP access without the SDK.

### Endpoint

```
POST https://[your-database-id].lite.bunnydb.net/v2/pipeline
```

### Authentication

```
Authorization: Bearer your-access-token
Content-Type: application/json
```

### Request Format

```json
{
  "requests": [
    { "type": "execute", "stmt": { "sql": "SELECT * FROM users" } },
    { "type": "close" }
  ]
}
```

### Parameter Binding

**Positional:**
```json
{
  "type": "execute",
  "stmt": {
    "sql": "SELECT * FROM users WHERE id = ?",
    "args": [{ "type": "integer", "value": "1" }]
  }
}
```

**Named:**
```json
{
  "type": "execute",
  "stmt": {
    "sql": "SELECT * FROM users WHERE name = :name",
    "named_args": [
      { "name": "name", "value": { "type": "text", "value": "Kit" } }
    ]
  }
}
```

### Value Types

| Type | Description |
|------|-------------|
| `null` | NULL value |
| `integer` | 64-bit signed integer (as string) |
| `float` | 64-bit float (as string) |
| `text` | UTF-8 string |
| `blob` | Base64-encoded binary |

### cURL Example

```bash
curl -X POST "https://my-db.lite.bunnydb.net/v2/pipeline" \
  -H "Authorization: Bearer your-access-token" \
  -H "Content-Type: application/json" \
  -d '{
    "requests": [
      { "type": "execute", "stmt": { "sql": "SELECT * FROM users" } },
      { "type": "close" }
    ]
  }'
```

---

## Limits (Public Preview)

| Resource | Limit |
|----------|-------|
| Max databases | 50 |
| Max database size | 1 GB |
| Interactive transaction timeout | 5 seconds |
| Data loss window (failover) | up to 10 seconds |

Limits may change as the product exits preview. Contact Bunny support for increases.

---

## Best Practices for Edge Scripts + Database

1. **Create the client at the top level** — outside the request handler — so it's reused across requests.
2. **Use parameterized queries** — never interpolate user input into SQL strings.
3. **Use batch transactions** for multi-statement writes — atomic and auto-rollback.
4. **Keep interactive transactions under 5 seconds** — they lock the database.
5. **Use `@libsql/client/web`** import path in Edge Scripts (not the default export).
6. **Account for eventual consistency** — if you write then immediately read, you might hit a stale replica. For critical reads-after-writes, consider structuring your response to return the written data directly rather than re-querying.
7. **Initialize schema idempotently** — use `CREATE TABLE IF NOT EXISTS` at startup.
8. **Mind the 50 subrequest limit** — each DB query counts as a subrequest from the edge script.
