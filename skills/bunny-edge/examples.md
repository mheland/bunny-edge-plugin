# Bunny Edge Scripting — Examples

## Standalone Scripts

### Return JSON

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http.serve(
  async (request: Request): Response | Promise<Response> => {
    const data = {
      weather: "sunny",
      temperature: 27,
      windspeed: 0,
      uvindex: 7,
    };

    const json = JSON.stringify(data);

    return new Response(json, {
      headers: {
        "content-type": "application/json",
      },
    });
  },
);
```

### Fetch/Proxy URL

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http.serve(
  async (request: Request): Response | Promise<Response> => {
    const url = new URL(request.url);
    const fetchUrl = "https://example.com";
    return fetch(fetchUrl + url.pathname);
  },
);
```

### Return HTML

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http.serve(
  async (request: Request): Response | Promise<Response> => {
    const html = `<!DOCTYPE html>
<html>
<head><title>Edge Page</title></head>
<body>
  <h1>Hello from the edge!</h1>
  <p>Request URL: ${request.url}</p>
</body>
</html>`;

    return new Response(html, {
      headers: { "content-type": "text/html; charset=utf-8" },
    });
  },
);
```

### REST API with Routing

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http.serve(async (request: Request): Response | Promise<Response> => {
  const url = new URL(request.url);
  const { pathname } = url;
  const method = request.method;

  // CORS headers
  const corsHeaders = {
    "Access-Control-Allow-Origin": "*",
    "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type",
  };

  if (method === "OPTIONS") {
    return new Response(null, { status: 204, headers: corsHeaders });
  }

  if (pathname === "/api/health" && method === "GET") {
    return new Response(JSON.stringify({ status: "ok" }), {
      headers: { ...corsHeaders, "content-type": "application/json" },
    });
  }

  if (pathname === "/api/echo" && method === "POST") {
    const body = await request.text();
    return new Response(body, {
      headers: { ...corsHeaders, "content-type": "application/json" },
    });
  }

  return new Response(JSON.stringify({ error: "Not Found" }), {
    status: 404,
    headers: { ...corsHeaders, "content-type": "application/json" },
  });
});
```

### Method-Restricted Endpoint

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http.serve(
  async (request: Request): Response | Promise<Response> => {
    if (request.method !== "GET") {
      return new Response("Not Allowed", {
        status: 405,
        headers: { Allow: "GET" },
      });
    }

    return new Response(JSON.stringify({ hello: "world" }), {
      headers: { "Content-type": "application/json" },
    });
  },
);
```

### WebSocket Echo Server

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http.serve(async (request: Request) => {
  const { response, socket } = request.upgradeWebSocket();

  socket.addEventListener("open", () => {
    console.log("Client connected");
  });

  socket.addEventListener("message", (event) => {
    socket.send(`Echo: ${event.data}`);
  });

  socket.addEventListener("close", (event) => {
    console.log("Client disconnected:", event.code);
  });

  return response;
});
```

---

## Middleware Scripts

### Modify Response Body (Title Injection)

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

async function onOriginResponse(
  context: { request: Request; response: Response },
): Promise<Response> | Response | void {
  let body = await context.response.text();
  body = body.replace("</title>", " - bunny.net</title>");

  const headers = new Headers(context.response.headers);
  headers.delete("Content-Length");
  headers.delete("Content-Encoding");

  return new Response(body, {
    status: context.response.status,
    headers,
  });
}

BunnySDK.net.http
  .servePullZone({ url: "https://your-origin.example.com" })
  .onOriginResponse(onOriginResponse);
```

### Auth Middleware

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http
  .servePullZone({ url: "https://your-origin.example.com" })
  .onOriginRequest(async (context) => {
    const authHeader = context.request.headers.get("Authorization");

    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return new Response(JSON.stringify({ error: "Unauthorized" }), {
        status: 401,
        headers: { "content-type": "application/json" },
      });
    }

    // Optionally validate the token, add user info to forwarded request
    const headers = new Headers(context.request.headers);
    headers.set("X-User-Verified", "true");

    return new Request(context.request.url, {
      method: context.request.method,
      headers,
      body: context.request.body,
    });
  });
```

### A/B Testing Middleware

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http
  .servePullZone({ url: "https://your-origin.example.com" })
  .onOriginRequest(async (context) => {
    const url = new URL(context.request.url);
    const abGroup = context.request.headers.get("X-AB") || "A";

    if (abGroup === "B") {
      // Route to variant origin
      return fetch(`https://variant-b.example.com${url.pathname}`);
    }

    return context.request;
  });
```

### Add Security Headers

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http
  .servePullZone({ url: "https://your-origin.example.com" })
  .onOriginResponse(async (context) => {
    const headers = new Headers(context.response.headers);
    headers.set("X-Content-Type-Options", "nosniff");
    headers.set("X-Frame-Options", "DENY");
    headers.set("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
    headers.set("Referrer-Policy", "strict-origin-when-cross-origin");

    return new Response(context.response.body, {
      status: context.response.status,
      headers,
    });
  });
```

---

## HTMLRewriter Examples

### Middleware with HTMLRewriter (Inject Banner)

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";

BunnySDK.net.http
  .servePullZone({ url: "https://your-origin.example.com" })
  .onOriginResponse(async (context) => {
    return new HTMLRewriter()
      .on("body", {
        element(el) {
          el.prepend(
            '<div style="background:yellow;padding:8px;text-align:center">Maintenance scheduled tonight</div>',
            { html: true },
          );
        },
      })
      .transform(context.response);
  });
```

### Rewrite All Links to HTTPS

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

### Remove Tracking Scripts

```ts
new HTMLRewriter()
  .on("script[src*='tracker']", {
    element(el) { el.remove(); },
  })
  .transform(response);
```

### Inject Analytics at End of Document

```ts
new HTMLRewriter()
  .onDocument({
    end(end) {
      end.append('<script src="/analytics.js"></script>', { html: true });
    },
  })
  .transform(response);
```

### Async Include (SSI-like)

```ts
new HTMLRewriter()
  .on("include[src]", {
    async element(el) {
      const src = el.getAttribute("src");
      const partial = await fetch(src);
      el.replace(partial.body, { html: true });
    },
  })
  .transform(response);
```

### Reusable Class-Based Handler

```ts
class AttributeRewriter {
  #attrName: string;
  constructor(attrName: string) { this.#attrName = attrName; }
  element(el: Element) {
    const value = el.getAttribute(this.#attrName);
    if (value) {
      el.setAttribute(this.#attrName, value.replace("http://", "https://"));
    }
  }
}

new HTMLRewriter()
  .on("a", new AttributeRewriter("href"))
  .on("img", new AttributeRewriter("src"))
  .transform(response);
```

---

## Database-Backed Scripts

### REST API with Bunny Database

```ts
import * as BunnySDK from "@bunny.net/edgescript-sdk";
import { createClient } from "@libsql/client/web";
import process from "node:process";

const client = createClient({
  url: process.env.BUNNY_DATABASE_URL,
  authToken: process.env.BUNNY_DATABASE_AUTH_TOKEN,
});

// Initialize schema (runs once at cold start)
await client.execute(`
  CREATE TABLE IF NOT EXISTS todos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    completed INTEGER DEFAULT 0,
    created_at TEXT DEFAULT (datetime('now'))
  )
`);

BunnySDK.net.http.serve(async (request: Request): Response | Promise<Response> => {
  const url = new URL(request.url);
  const headers = { "content-type": "application/json" };

  // GET /todos — list all
  if (url.pathname === "/todos" && request.method === "GET") {
    const result = await client.execute("SELECT * FROM todos ORDER BY created_at DESC");
    return new Response(JSON.stringify(result.rows), { headers });
  }

  // POST /todos — create
  if (url.pathname === "/todos" && request.method === "POST") {
    const body = await request.json();
    const result = await client.execute({
      sql: "INSERT INTO todos (title) VALUES (?)",
      args: [body.title],
    });
    return new Response(JSON.stringify({ id: Number(result.lastInsertRowid) }), {
      status: 201, headers,
    });
  }

  // PATCH /todos/:id — toggle complete
  if (url.pathname.match(/^\/todos\/\d+$/) && request.method === "PATCH") {
    const id = url.pathname.split("/")[2];
    await client.execute({
      sql: "UPDATE todos SET completed = NOT completed WHERE id = ?",
      args: [id],
    });
    return new Response(JSON.stringify({ ok: true }), { headers });
  }

  // DELETE /todos/:id
  if (url.pathname.match(/^\/todos\/\d+$/) && request.method === "DELETE") {
    const id = url.pathname.split("/")[2];
    await client.execute({
      sql: "DELETE FROM todos WHERE id = ?",
      args: [id],
    });
    return new Response(null, { status: 204 });
  }

  return new Response(JSON.stringify({ error: "Not Found" }), { status: 404, headers });
});
```

### Batch Transaction Example

```ts
import { createClient } from "@libsql/client/web";
import process from "node:process";

const client = createClient({
  url: process.env.BUNNY_DATABASE_URL,
  authToken: process.env.BUNNY_DATABASE_AUTH_TOKEN,
});

// Atomic batch — all succeed or all roll back
const results = await client.batch([
  {
    sql: "UPDATE accounts SET balance = balance - ? WHERE id = ?",
    args: [100, 1],
  },
  {
    sql: "UPDATE accounts SET balance = balance + ? WHERE id = ?",
    args: [100, 2],
  },
  {
    sql: "INSERT INTO transfers (from_id, to_id, amount) VALUES (?, ?, ?)",
    args: [1, 2, 100],
  },
], "write");
```

### Interactive Transaction with Validation

```ts
const tx = await client.transaction("write");
try {
  const result = await tx.execute({
    sql: "SELECT balance FROM accounts WHERE id = ?",
    args: [1],
  });

  const balance = result.rows[0].balance as number;
  if (balance < 100) {
    await tx.rollback();
    return new Response(JSON.stringify({ error: "Insufficient funds" }), {
      status: 400,
      headers: { "content-type": "application/json" },
    });
  }

  await tx.execute({
    sql: "UPDATE accounts SET balance = balance - 100 WHERE id = ?",
    args: [1],
  });
  await tx.commit();

  return new Response(JSON.stringify({ ok: true }), {
    headers: { "content-type": "application/json" },
  });
} catch (e) {
  await tx.rollback();
  throw e;
}
```
