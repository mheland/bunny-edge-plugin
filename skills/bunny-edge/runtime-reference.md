# Bunny Edge Scripting — Runtime Reference

## HTMLRewriter API

Streaming, selector-based HTML rewriting at the edge. No need to buffer the entire document.

### Constructor

```ts
new HTMLRewriter(options?)
```

Options:
- `enableEsiTags` (boolean, default `false`) — activates ESI tag processing.

### Methods

**`on(selector, handlers)`** — Register element-level handlers matching a CSS selector. Returns `this` (chainable).

Handler object shape:
```ts
{
  element?(el: Element): void | Promise<void>;
  comments?(comment: Comment): void | Promise<void>;
  text?(text: TextChunk): void | Promise<void>;
}
```

**`onDocument(handlers)`** — Register document-level handlers. Returns `this` (chainable).

Handler object shape:
```ts
{
  doctype?(doctype: Doctype): void | Promise<void>;
  comments?(comment: Comment): void | Promise<void>;
  text?(text: TextChunk): void | Promise<void>;
  end?(end: DocumentEnd): void | Promise<void>;
}
```

**`transform(response: Response)`** — Apply all registered handlers and return a new `Response` with a streaming transformed body. Strips `Content-Length` header automatically.

### Element

Properties:
- `tagName` (read/write)
- `namespaceURI` (readonly)
- `attributes` (readonly iterable)
- `removed` (readonly boolean)

Attribute methods:
- `getAttribute(name): string | null`
- `hasAttribute(name): boolean`
- `setAttribute(name, value): Element`
- `removeAttribute(name): Element`

Content mutation methods (all accept `string | ReadableStream | Response`, return `Element`):
- `before(content, options?)` — insert before opening tag
- `after(content, options?)` — insert after closing tag
- `prepend(content, options?)` — insert after opening tag (inside element)
- `append(content, options?)` — insert before closing tag (inside element)
- `replace(content, options?)` — replace entire element
- `setInnerContent(content, options?)` — replace inner content only

Removal:
- `remove()` — remove element and its content
- `removeAndKeepContent()` — remove tags, keep children

End tag:
- `onEndTag(handler: (tag: EndTag) => void | Promise<void>)` — execute when closing tag is encountered

Content options: `{ html: true }` inserts as raw HTML; `{ html: false }` (default) escapes as text.

### Comment

Properties:
- `text` (read/write)
- `removed` (readonly)

Methods: `before()`, `after()`, `replace()`, `remove()` — same signature as Element mutations.

### TextChunk

Properties:
- `text` (readonly string)
- `lastInTextNode` (readonly boolean)
- `removed` (readonly)

Methods: `before()`, `after()`, `replace()`, `remove()`.

### EndTag

Properties:
- `name` (read/write)

Methods: `before()`, `after()`, `remove()`.

### Doctype (readonly)

Properties: `name`, `publicId`, `systemId`.

### DocumentEnd

Methods:
- `append(content: string, options?)` — strings only, no streams.

### Supported CSS Selectors

- Type: `div`, `p`, `a`
- Class: `.class`
- ID: `#id`
- Attribute: `[attr]`, `[attr=value]`, `[attr~=value]`, `[attr^=value]`, `[attr$=value]`, `[attr*=value]`, `[attr|=value]`
- Pseudo-classes: `:nth-child()`, `:first-child`, `:nth-of-type()`, `:first-of-type`, `:not()`
- Combinators: descendant (`E F`), child (`E > F`)

### HTMLRewriter Examples

**Rewrite all links to HTTPS:**
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

**Inject analytics script:**
```ts
new HTMLRewriter()
  .onDocument({
    end(end) {
      end.append('<script src="/analytics.js"></script>', { html: true });
    },
  })
  .transform(response);
```

**Remove tracking scripts:**
```ts
new HTMLRewriter()
  .on("script[src*='tracker']", {
    element(el) { el.remove(); },
  })
  .transform(response);
```

**Async element replacement (e.g., SSI-like includes):**
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

**Class-based handler for reusability:**
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

## WebSockets

Low-latency bidirectional communication at the edge.

**Prerequisite:** Enable WebSockets on the Pull Zone (Pull Zone > General > WebSockets).

### Upgrading a Request

```ts
const { response, socket } = request.upgradeWebSocket(options?);
```

Options:
- `protocol` (string) — WebSocket subprotocol (e.g., `"graphql-ws"`)
- `idleTimeout` (number, default 30) — seconds before closing idle connections

Returns:
- `response` — the HTTP upgrade response (return this from your handler)
- `socket` — the WebSocket interface

### Socket API

Methods:
- `socket.send(data)` — send string, JSON, ArrayBuffer, Blob, or ArrayBufferView
- `socket.close(code?, reason?)` — close with optional code and reason
- `socket.addEventListener(type, listener, options?)` — listen for events

Events:
- `open` — connection established
- `message` — data received (`event.data`)
- `close` — connection closed (`event.code`, `event.reason`, `event.wasClean`)
- `error` — error occurred

### Echo Server Example

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

## Node:FS — Virtual File System

Sandboxed file system with Node.js compatibility.

### Writable Paths

| Path | Access |
|------|--------|
| `/home/user/` | Read/Write (cwd) |
| `/tmp/` | Read/Write |

### Constraints

- Max path length: 4,096 characters
- Max directory depth: 48 segments
- Storage capacity: 64 MB per script
- All files open in `w+` mode

### Supported Operations

```ts
import * as fs from "node:fs/promises";

// Read/write files
await fs.writeFile("/tmp/data.json", JSON.stringify(obj), "utf-8");
const content = await fs.readFile("/tmp/data.json", "utf-8");

// Directory operations
await fs.mkdir("/tmp/myapp/data", { recursive: true });
const files = await fs.readdir("/tmp/myapp/data");
await fs.rmdir("/tmp/myapp/data");

// File operations
await fs.copyFile("/tmp/source.txt", "/tmp/dest.txt");
await fs.rename("/tmp/old.txt", "/tmp/new.txt");
await fs.unlink("/tmp/remove-me.txt");

// Existence check
try {
  await fs.access("/tmp/file.txt");
} catch (e) {
  if (e.code === "ENOENT") { /* does not exist */ }
}

// Stat (only size and type are reliable)
const stat = await fs.stat("/tmp/file.txt");
stat.size;
stat.isFile();
stat.isDirectory();

// Chunked reading
const fh = await fs.open("/tmp/large.bin", "r");
const buf = new Uint8Array(1024);
const { bytesRead } = await fh.read(buf, 0, 1024, 0);
await fh.close();
```

### Not Supported

- Symbolic/hard links (`symlink`, `link`) — use `copyFile` instead
- Timestamps (`atime`, `mtime`, etc.) — unreliable values
- Permissions (`chmod`, `chown`) — no practical effect (all files are 644)
- File watching (`watch`, `watchFile`)

---

## Environment Variables

Set via dashboard: Script > Env Configuration > Environment Variables.

Access in code:
```ts
// Deno style
const value = Deno.env.get("MY_VAR");

// Node.js style
import process from "node:process";
const value = process.env.MY_VAR;
```

Limits: 128 variables per script, 2 KB max per variable.

---

## Secrets

Set via dashboard: Script > Env Configuration > Environment Secrets. Encrypted — cannot be viewed after creation, only updated or deleted.

Access identically to environment variables:
```ts
const apiKey = Deno.env.get("API_KEY");
```

**Important:** Variable and secret names must be unique within the same script.
