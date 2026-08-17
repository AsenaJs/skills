# Controllers & Context

How to define HTTP routes with decorators and handle requests/responses through Asena's unified Context API, plus static file serving and frontend HTML controllers (as of @asenajs/asena 0.10.x, ergenecore / hono-adapter 3.x).

## Contents

- Controller quick start
- Request data: params, query, body
- Responses
- Streaming
- Cookies and headers
- Per-request state
- Static files (@StaticServe)
- Frontend controller (@FrontendController)

## Controller quick start

Controllers stay thin: parse the request, delegate to a service, shape the response. Business logic, repository/database access, and multi-service orchestration belong in services — never in the controller.

```typescript
import { Controller } from '@asenajs/asena/decorators';
import { Get, Post } from '@asenajs/asena/decorators/http';
import type { Context } from '@asenajs/ergenecore';

@Controller('/users')
export class UserController {
  @Get('/')
  async list(context: Context) {
    const page = (await context.getQuery('page')) || '1';
    return context.send({ users: [], page });
  }

  @Get('/:id')
  async getById(context: Context) {
    const id = context.getParam('id');
    return context.send({ id });
  }

  @Post('/')
  async create(context: Context) {
    const body = await context.getBody();
    return context.send({ created: true, data: body }, 201);
  }
}
```

**Note:** Both adapters expose the identical Context API — only the type import differs (`@asenajs/hono-adapter` instead of `@asenajs/ergenecore`).

### @Controller options

```typescript
@Controller('/api/users')                                      // string shorthand
@Controller({ path: '/admin', middlewares: [AuthMiddleware] }) // full form
```

| Property | Type | Required | Description |
|:---------|:-----|:---------|:------------|
| `path` | `string` | No — defaults to `/` (bare `@Controller()` is valid; the CLI scaffolds it) | Base path prefixed to every route |
| `middlewares` | `Middleware[]` | No | Applied to all routes in the controller |

### Route decorators

`@Get` `@Post` `@Put` `@Patch` `@Delete` `@Options` `@Head` — all from `@asenajs/asena/decorators/http`. Two extras from the same subpath: `@All(path)` handles every HTTP method on one route; `@Route(method, path)` handles a custom verb the standard decorators don't cover (e.g. `'PURGE'`). Each accepts a path string or an options object:

```typescript
@Post({
  path: '/',
  middlewares: [AuthMiddleware],   // route-level middleware
  validator: CreateUserValidator,  // validation middleware class
})
async create(context: Context) { /* ... */ }
```

`@Get` additionally accepts `staticServe: <StaticServeService subclass>` (see Static files below).

Execution order: global middleware → controller middleware → route middleware → handler.

**Warning:** `await` every Promise in handlers — an unawaited Promise that rejects shuts the server down.

### Injecting services

```typescript
import { Inject } from '@asenajs/asena/decorators/ioc';

@Controller('/users')
export class UserController {
  @Inject(UserService)
  private userService: UserService;
}
```

String form `@Inject('UserService')` also resolves; give the service an explicit name (`@Service('UserService')`) so the Bun bundler's class renaming in production builds cannot change the container key.

## Request data: params, query, body

**`getParam()` is synchronous; `getQuery`, `getQueryAll`, `getBody` and all other body/cookie helpers return Promises — always `await` them.** Forgetting `await` silently yields the Promise object, not the value:

```typescript
const page = context.getQuery('page') || '1';          // WRONG: page is a Promise, || never applies
const page = (await context.getQuery('page')) || '1';  // correct
```

A missing query key resolves to `''` on ergenecore and `undefined` on hono-adapter (despite the `Promise<string>` type) — treat falsy as absent, never compare against one adapter's miss value.

```typescript
@Get('/:userId/posts/:postId')
async getPost(context: Context) {
  const userId = context.getParam('userId');      // sync, always string — Number() to convert
  const q = await context.getQuery('q');          // ?q=asena
  const tags = await context.getQueryAll('tags'); // ?tags=a&tags=b → string[]
  const all = context.getAllQueries();            // sync: { q: 'asena', tags: ['a', 'b'] }
  return context.send({ userId, q, tags, all });
}

@Post('/users')
async create(context: Context) {
  const body = await context.getBody<{ name: string; email: string }>(); // typed JSON
  return context.send({ created: true, user: body }, 201);
}
```

**Warning:** Empty request body — ergenecore returns `{}`; hono may throw, so wrap `getBody()` in try/catch on hono.

Other body readers: `getParseBody()` (auto-detects JSON/form-data/URL-encoded by Content-Type), `getFormData()`, `getArrayBuffer()`, `getBlob()`.

```typescript
const formData = await context.getFormData();
const file = formData.get('file');               // File instance for uploads
if (!file || !(file instanceof File)) return context.send({ error: 'No file' }, 400);
```

### Request method reference

| Method | Return type |
|:-------|:------------|
| `getParam(name)` | `string` (sync) |
| `getQuery(name)` | `Promise<string>` |
| `getQueryAll(name)` | `Promise<string[]>` |
| `getAllQueries()` | `Record<string, string \| string[]>` (sync) |
| `getBody<T>()` | `Promise<T>` |
| `getParseBody()` | `Promise<any>` |
| `getFormData()` | `Promise<FormData>` |
| `getArrayBuffer()` | `Promise<ArrayBuffer>` |
| `getBlob()` | `Promise<Blob>` |
| `getRequestIp()` | `string \| null` (sync, lazy + cached) |

## Responses

`send(data, statusOrOptions?)` auto-sets the content type and returns a `Response` — always `return` it.

```typescript
return context.send({ ok: true });                // 200 JSON
return context.send({ created: true }, 201);      // custom status
return context.send({ error: 'Not found' }, 404);
return context.send({ data }, { status: 200, headers: { 'X-Request-ID': crypto.randomUUID() } });
return context.html('<h1>Hello</h1>');            // text/html
return context.redirect('/login');                // 302 Found
```

| Method | Returns | Description |
|:-------|:--------|:------------|
| `send(data, statusOrOptions?)` | `Response` | JSON/text with auto content-type |
| `html(data, statusOrOptions?)` | `Response` | HTML response |
| `redirect(url)` | `Response` | 302 redirect |
| `setResponseHeader(key, value)` | `void` | Header merged into the final response; also carries through to streaming responses, usable from middleware |

Direct header access also works: `context.res.headers.set('X-My-Header', 'v')`.

**Warning:** Calling `setResponseHeader()` twice with the same key **replaces** the value on ergenecore but **appends** a second header on hono.

## Streaming

Three methods, identical on both adapters. Streams auto-close when the callback returns; call `stream.close()` only to end early.

| Method | Content-Type |
|:-------|:-------------|
| `stream(cb, onError?)` | none — binary, CSV, custom formats |
| `streamText(cb, onError?)` | `text/plain` |
| `streamSSE(cb, onError?)` | `text/event-stream` (+ `Cache-Control: no-cache`, `Connection: keep-alive`) |

```typescript
@Get('/events')
async events(context: Context) {
  return context.streamSSE(
    async (stream) => {
      stream.onAbort(() => { /* client disconnected — clean up */ });
      while (!stream.aborted) {
        await stream.writeSSE({ data: JSON.stringify({ t: Date.now() }), event: 'update', id: '1' });
        await Bun.sleep(1000);
      }
    },
    async (error, stream) => {
      await stream.writeSSE({ data: JSON.stringify({ error: error.message }), event: 'error' });
    },
  );
}

@Get('/export')
async export(context: Context) {
  return context.stream(async (stream) => {
    await stream.writeln('name,email');
    await stream.pipe(someReadableStream); // pipe any ReadableStream
  });
}
```

StreamWriter API: `write(input: Uint8Array | string)`, `writeln(input: string)`, `close()`, `pipe(body: ReadableStream)`, `onAbort(listener)`, `aborted: boolean`, `closed: boolean`. `streamSSE` adds `writeSSE(message: SSEMessage)`:

```typescript
interface SSEMessage {
  data: string;    // multi-line strings auto-split into separate data: lines
  event?: string;  // event type name
  id?: string;     // event ID for reconnection
  retry?: number;  // reconnection time in ms
}
```

## Cookies and headers

### Request headers

`context.headers` is a plain `Record<string, string>` on both adapters:

```typescript
const token = context.headers['authorization'];
```

The raw `context.req` differs: a native `Request` on ergenecore (`context.req.headers.get('x-server')`), a `HonoRequest` on hono where `header()` is a function (`context.req.header()['X-Server']`).

### Cookies

All cookie helpers are async.

```typescript
const session = await context.getCookie('session');  // string | false
await context.setCookie('theme', 'dark', {
  extraOptions: { path: '/', maxAge: 86400 * 30, httpOnly: false, sameSite: 'lax' },
});
await context.deleteCookie('session');

// Signed cookies — tamper-proof
await context.setCookie('userId', 'u1', {
  secret: 'your-secret-key',
  extraOptions: { httpOnly: true, secure: true, maxAge: 3600 },
});
const userId = await context.getCookie('userId', 'your-secret-key'); // false when invalid
```

## Per-request state

`setValue`/`getValue` (both sync) share data between middleware and handlers:

```typescript
context.setValue('user', user);        // in middleware
const user = context.getValue('user'); // in handler
```

Type them by augmenting `AsenaVariables` — the module path must be exactly `'@asenajs/asena/adapter'` and the declaration file must be in your TS compilation:

```typescript
declare module '@asenajs/asena/adapter' {
  interface AsenaVariables {
    user: User;
    requestId: string;
  }
}
```

Generic form still works for dynamic keys: `context.getValue<string>('dynamicKey')`.

WebSocket upgrades: call `setWebSocketValue(value)` in pre-upgrade middleware; the adapter injects it into `ws.data.values` automatically, so `getWebSocketValue<T>()` is rarely needed.

## Static files (@StaticServe)

Extend the adapter's `StaticServeService`, decorate with `@StaticServe({ root })`, then attach via the `staticServe` route option:

```typescript
import { StaticServe } from '@asenajs/asena/decorators';
import { StaticServeService, type Context } from '@asenajs/ergenecore';
import path from 'path';

@StaticServe({ root: path.join(process.cwd(), 'public') }) // root must be absolute, never './public'
export class StaticServeMiddleware extends StaticServeService {
  public rewriteRequestPath(reqPath: string): string {
    return reqPath.replace(/^\/static\/|^\/static/, '');   // strip the route prefix
  }
}

@Controller({ path: '/static' })
export class StaticController {
  @Get({ path: '/*', staticServe: StaticServeMiddleware })
  public static() {}
}
// GET /static/style.css → public/style.css
```

`@StaticServe` options: only `root: string` (absolute directory containing the files).

### Lifecycle hooks (all optional — override only what you need)

| Hook | Signature | Purpose |
|:-----|:----------|:--------|
| `rewriteRequestPath` | `(reqPath: string) => string` | Transform the path before file lookup — strip prefixes, SPA fallback. Strip `..` here to block directory traversal. |
| `onFound` | `(filePath: string, c: Context) => void \| Promise<void>` | Runs when a file is served — logging/analytics. `filePath` is not absolute and differs per adapter: ergenecore passes the rewritten request path, hono the root-joined path (`./public/index.html`). |
| `onNotFound` | `(reqPath: string, c: Context) => void \| Promise<void>` | Observational — return value discarded, adapter still answers 404. Mark with `@Override()` (from `@asenajs/asena/decorators`) to make ergenecore skip its 404 and fall through to route handlers. |

**Warning:** `c.setResponseHeader()` inside `onFound` is silently dropped on ergenecore (static responses are built directly from the file; it works on hono). Use the portable `extra` property instead:

```typescript
@StaticServe({ root: path.join(process.cwd(), 'public') })
export class PublicStaticServe extends StaticServeService {
  public extra = {
    cacheControl: 'public, max-age=31536000, immutable',
    headers: { 'X-Served-By': 'Asena Static Serve' },
    precompressed: true,                 // prefer .br/.gz variants when present
    mimes: { '.ts': 'text/typescript' }, // override MIME detection
  };
}
```

SPA fallback pattern: in `rewriteRequestPath`, when the resolved file does not exist and the path has no extension, return `'index.html'`.

## Frontend controller (@FrontendController)

Serves HTML pages via Bun's native HTML imports. Routes register directly on `Bun.serve()`'s `routes` option — **the entire middleware chain is bypassed** (no CORS, auth, rate limiting, logging). Use a regular `@Controller` when you need middleware; for SPAs, handle auth client-side against your API.

```typescript
import { FrontendController } from '@asenajs/asena/decorators';
import { Page } from '@asenajs/asena/decorators/http';

@FrontendController('/app')             // or { path: '/app', name: 'AppFrontend' }
export class AppFrontendController {
  @Page('/')                            // → /app
  public home() { return import('./pages/home.html'); }

  @Page('/settings')                    // → /app/settings
  public settings() { return import('./pages/settings.html'); }
}
```

Each `@Page` method must return an HTML `import()`; Bun bundles linked CSS/JS/TS automatically.

**Build:** `asena build` rewrites HTML import paths, but you must copy the files with `include` in `asena-config.ts` — without it the imports fail at runtime in production:

```typescript
import { defineConfig } from '@asenajs/asena-cli';

export default defineConfig({
  sourceFolder: 'src',
  rootFile: 'src/index.ts',
  include: ['src/frontend/pages'],
  buildOptions: { outdir: 'dist' },
});
```

**Warning:** Never add `'*.html'` to `buildOptions.external` — it breaks the CLI's HTML import path rewriting.
