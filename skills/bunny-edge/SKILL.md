---
name: bunny-edge
description: Scaffold, develop, and deploy Bunny CDN Edge Scripts with edge-replicated database. Use when working with Bunny edge scripting, edge functions, CDN middleware, HTMLRewriter, Bunny Database, or deploying scripts to Bunny's edge network.
argument-hint: "[scaffold|deploy|middleware|rewriter] [description]"
---

# Bunny CDN Edge Scripting

You are an expert in Bunny CDN Edge Scripting. Help the user scaffold, develop locally, and deploy edge scripts to Bunny's platform.

For full API details, runtime reference, and examples beyond what's covered here, see the supporting files in this skill directory:
- [runtime-reference.md](runtime-reference.md) — HTMLRewriter API, WebSockets, Node:FS, environment variables, secrets
- [database-reference.md](database-reference.md) — Bunny Database (edge-replicated libSQL), SDK, SQL API, transactions, replication model
- [deployment-guide.md](deployment-guide.md) — GitHub integration, Actions workflow, API endpoints, limits, pricing
- [examples.md](examples.md) — Complete code examples for standalone and middleware scripts

## Architecture Overview

Bunny Edge Scripts run on **Deno + V8** at the CDN edge. There are two types:

- **Standalone scripts** — Handle HTTP requests directly without an origin server. Use `BunnySDK.net.http.serve()`.
- **Middleware scripts** — Intercept/modify requests and responses flowing through a Pull Zone. Use `BunnySDK.net.http.servePullZone()`. Can be attached via Edge Rules.

## Scaffolding a New Project

When the user asks to scaffold/create a new edge script project:

### Standalone Script

Create this file structure:
```
project-name/
  script.ts
  .github/
    workflows/
      deploy.yml
```

**script.ts** — starter template:
```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http.serve(async (request: Request): Response | Promise<Response> => {
  const url = new URL(request.url);

  // Route handling
  if (url.pathname === "/") {
    return new Response("Hello from the edge!", {
      headers: { "content-type": "text/plain" },
    });
  }

  return new Response("Not Found", { status: 404 });
});
```

### Middleware Script

**script.ts** — starter template:
```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http.servePullZone(
  // In local dev, specify the origin URL. In production, the Pull Zone origin is used automatically.
  { url: "https://your-origin.example.com" },
)
  .onOriginRequest(async (context) => {
    // Modify request before it reaches origin
    // Return a Request to modify, or a Response to short-circuit
    return context.request;
  })
  .onOriginResponse(async (context) => {
    // Modify response before it reaches the client and cache
    return context.response;
  });
```

### GitHub Actions Workflow

**.github/workflows/deploy.yml**:
```yml
name: Deploy to Bunny Edge

on:
  push:
    branches:
      - "main"

jobs:
  publish:
    runs-on: ubuntu-latest
    name: "Deploy Edge Script"
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Deploy Script to Bunny Edge Scripting
        uses: BunnyWay/actions/deploy-script@main
        with:
          script_id: ${{ secrets.SCRIPT_ID }}
          deploy_key: ${{ secrets.DEPLOY_KEY }}
          file: "script.ts"
```

## Local Development

Scripts run locally with Deno:
```bash
deno run -A script.ts
```

This starts a local server. Test with curl or a browser. For middleware scripts, the `url` option in `servePullZone()` is used as the origin during local development.

## Deployment Workflow

**IMPORTANT: GitHub repo creation constraint.** You cannot link an existing GitHub repo to a Bunny edge script. The repo must be created from Bunny's side first:

1. Go to **Edge Platform > Scripting > Add Script** in the Bunny dashboard
2. Choose **"Deploy and edit with GitHub"**
3. Select script type (Standalone or Middleware)
4. Either create a new repo from a Bunny template, or connect an existing repo and configure:
   - Entry file location (e.g., `script.ts`)
   - Build/install commands (if any)
5. Bunny creates/configures the GitHub Actions deployment pipeline
6. Get **Script ID** and **Deploy Key** from Script > Deployments > Settings
7. Add `SCRIPT_ID` and `DEPLOY_KEY` as GitHub repository secrets
8. Push to `main` to trigger deployment

Alternatively, deploy via the Bunny API:
```bash
# Upload code
curl -X POST "https://api.bunny.net/compute/script/{id}/code" \
  -H "AccessKey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"Code": "..."}'

# Publish release
curl -X POST "https://api.bunny.net/compute/script/{id}/publish" \
  -H "AccessKey: YOUR_API_KEY"
```

## Key APIs

### Standalone — `BunnySDK.net.http.serve(handler)`
```ts
serve(handler: (request: Request) => Response | Promise<Response>)
```
Receives standard Web API `Request`, must return `Response`.

### Middleware — `BunnySDK.net.http.servePullZone(options)`
```ts
servePullZone(options: { url: string }): PullZoneHandler
```
Returns a chainable handler with:
- `.onOriginRequest(handler)` — intercept before origin. Return `Request` to modify, `Response` to short-circuit.
- `.onOriginResponse(handler)` — intercept before client/cache. Receives `{ request, response }`, return modified `Response`.

### HTMLRewriter
Streaming HTML transformation at the edge — no need to buffer the entire document:
```ts
new HTMLRewriter()
  .on("a[href]", {
    element(el) {
      const href = el.getAttribute("href");
      if (href?.startsWith("http://")) {
        el.setAttribute("href", href.replace("http://", "https://"));
      }
    },
  })
  .transform(response);
```

See [runtime-reference.md](runtime-reference.md) for full HTMLRewriter API.

### WebSockets
```ts
const { response, socket } = request.upgradeWebSocket();
socket.addEventListener("message", (event) => {
  socket.send(`Echo: ${event.data}`);
});
return response;
```
Requires enabling WebSockets on the Pull Zone: Pull Zone > General > WebSockets.

### Environment Variables & Secrets
```ts
// Environment variable
const value = Deno.env.get("MY_VAR");
// or
import process from "node:process";
const value = process.env.MY_VAR;
```
Set via dashboard: Script > Env Configuration. Use **Secrets** for sensitive values (API keys, tokens) — they are encrypted and cannot be viewed after creation.

### Bunny Database (Edge-Replicated)

Fully managed libSQL (SQLite fork) database with global replication. Databases and edge scripts are deployed to the same regions for minimal latency.

**Connect from Edge Scripts:**
1. Dashboard > Database > [DB] > Access > Generate Tokens > **Add Secrets to Edge Script**
2. This injects `BUNNY_DATABASE_URL` and `BUNNY_DATABASE_AUTH_TOKEN` as secrets

```ts
import { createClient } from "@libsql/client/web";
import process from "node:process";

const client = createClient({
  url: process.env.BUNNY_DATABASE_URL,
  authToken: process.env.BUNNY_DATABASE_AUTH_TOKEN,
});

// Parameterized query
const result = await client.execute({
  sql: "SELECT * FROM users WHERE id = ?",
  args: [1],
});

// Batch transaction (atomic, auto-rollback on failure)
await client.batch([
  { sql: "INSERT INTO users (name) VALUES (?)", args: ["Kit"] },
  { sql: "INSERT INTO users (name) VALUES (?)", args: ["Sam"] },
], "write");
```

**Key details:**
- Use `@libsql/client/web` import path (not default) in edge scripts
- Create client at top level (outside handler) for reuse
- Eventual consistency — replicas may lag; no read-your-writes guarantee on replicas
- Each DB query counts toward the 50 subrequest limit
- Interactive transactions have a 5-second timeout
- Max 50 databases, 1 GB each (Public Preview limits)

See [database-reference.md](database-reference.md) for full SDK, SQL API, replication architecture, and transactions.

### Node:FS (Virtual File System)
Sandboxed FS with 64MB storage. Writable paths: `/home/user/` and `/tmp/`.
```ts
import * as fs from "node:fs/promises";
await fs.writeFile("/tmp/cache.json", JSON.stringify(data), "utf-8");
const content = await fs.readFile("/tmp/cache.json", "utf-8");
```

## Limits

| Resource | Limit |
|----------|-------|
| CPU time per request | 30 seconds |
| Active memory | 128 MB |
| Subrequests per request | 50 |
| Script size | 10 MB |
| Startup time | 500 ms |
| Env variables per script | 128 |
| Env variable size | 2 KB |
| Virtual FS storage | 64 MB |

CPU time excludes I/O wait (fetch, etc.). Platform is forgiving for infrequent spikes.

## Pricing

- **CPU time**: $0.02 / 1,000s
- **Requests**: $0.20 / 1M requests
- CDN bandwidth billed separately

## Best Practices

1. **Use TypeScript** — the runtime supports it natively via Deno.
2. **Keep scripts small** — they run at the edge, aim for fast cold starts under 500ms.
3. **Use `HTMLRewriter` for HTML transforms** — it streams, avoiding buffering the full document.
4. **Use middleware for CDN augmentation** — auth, headers, A/B testing, content transforms.
5. **Use standalone for APIs/services** — no origin server needed.
6. **Offload config to env vars/secrets** — never hardcode API keys.
7. **Mind subrequest limits** — max 50 fetches per request.
8. **For middleware body modification** — delete `Content-Length` and `Content-Encoding` headers when modifying the response body.
