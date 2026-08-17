# Adapters

Choosing and configuring AsenaJS HTTP adapters: `@asenajs/ergenecore` (native Bun) and `@asenajs/hono-adapter`.

## Contents
- Choosing an Adapter
- Bootstrap & Factory Signatures
- Ergenecore Options
- Hono Adapter Options
- Adapter Base Classes
- Built-in Middleware
- Error & Not-Found Hooks
- Hono-Specific Features
- Migrating Between Adapters / From Standalone Hono
- Custom Adapters

## Choosing an Adapter

**Ergenecore is the default for new projects; choose hono-adapter when migrating from Hono or reusing the Hono middleware ecosystem.**

| | Ergenecore | Hono adapter |
|---|---|---|
| Implementation | `Bun.serve()` + native Bun APIs, SIMD routing, zero-copy `Bun.file()` | Hono framework under the hood |
| Runtime deps | Zero (`zod` is a peer) | `hono` + `zod` as peers |
| Runtime | Bun-exclusive | Bun-exclusive too — the adapter binds to `Bun.serve()`; Hono the framework is portable, this adapter is not |
| Ecosystem | Built-ins only | Full Hono middleware via `@Override` |

Benchmark: ~295k vs ~233k req/s (12 threads, 400 connections, 120s, hello-world).

```bash
bun add @asenajs/ergenecore zod            # ergenecore
bun add @asenajs/hono-adapter hono zod     # hono adapter
```

`zod` (>= 4.3.6) and `hono` (>= 4.12.9) are **peer dependencies** — your project owns them. Never let two copies of `hono` resolve (breaks `HTTPException` matching). Both adapters need Bun >= 1.3.12 and `@asenajs/asena` >= 0.10.0.

## Bootstrap & Factory Signatures

`createErgenecoreAdapter(options = {})` returns the adapter **alone**. `createHonoAdapter({ logger })` returns a **tuple** `[adapter, logger]` and **requires a logger** — calling it with no argument yields `[adapter, undefined]` and `AsenaServerFactory.create` then throws on the undefined logger.

```typescript
import { AsenaServerFactory } from '@asenajs/asena';
import { createErgenecoreAdapter } from '@asenajs/ergenecore';
import { logger } from './logger';

const adapter = createErgenecoreAdapter();

const server = await AsenaServerFactory.create({ adapter, logger, port: 3000 });
await server.start();
```

Hono variant of the adapter line:

```typescript
import { createHonoAdapter } from '@asenajs/hono-adapter';
import { AsenaLogger } from '@asenajs/asena-logger';

const [adapter, logger] = createHonoAdapter({ logger: new AsenaLogger() });
// legacy form: createHonoAdapter(myLogger)
```

`AsenaServerFactory.create()` options: `adapter`, `logger`, `port?`, `components?` (register only listed classes — skips project scan, use in tests), `overrides?` (replace components with test doubles by service name), `headless?` (boot without adapter), `health?` (`{ port, path? }`, path defaults to `/healthz`), `shutdown?` (signal handling / `@OnStop` timeouts), `keepAlive?` (default true in headless mode, false otherwise), `gc?` (`boolean` — run `Bun.gc(true)` once after startup to release bootstrap garbage).

## Ergenecore Options

| Option | Type | Default | Description |
|---|---|---|---|
| `port` | `number` | — | **Ignored** — `AsenaServer.start()` overwrites it with the port from `AsenaServerFactory.create({ port })` |
| `hostname` | `string` | `undefined` | The **only** way to set the hostname (`serveOptions.hostname` is overwritten) |
| `logger` | `ServerLogger` | `undefined` | Custom logger |
| `enableWebSocket` | `boolean` | `true` | WebSocket support |
| `websocketAdapter` | `ErgenecoreWebsocketAdapter` | auto | Custom WebSocket adapter |
| `logErrors` | `boolean` | `true` | Log framework-answered failures: 5xx at `error` with stack, 4xx at `debug` (fallback `info`) without, unmatched route at `info`. Requests your `onError`/`onNotFound` answered write nothing. `false` silences all three |

Two aliases exist; prefer the plain factory:

- `createProductionAdapter(options?)` — forwards to `createErgenecoreAdapter` with `enableWebSocket` defaulted to `true` (already the base default); no extra tuning.
- `createDevelopmentAdapter(options?)` — same, but WebSocket is **forced** on (`enableWebSocket: false` has no effect).

## Hono Adapter Options

`createHonoAdapter(loggerOrOptions)` — bare logger (legacy) or options object (recommended). Returns `[adapter, logger]`.

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| `logger` | `ServerLogger` | Yes | — | Logger instance |
| `app` | `Hono` | No | — | Pre-configured Hono app |
| `websocketAdapter` | `HonoWebsocketAdapter` | No | — | Custom WebSocket adapter |
| `strict` | `boolean` | No | `true` | Strict trailing-slash matching: `/users` and `/users/` are **different** routes. Set `false` behind reverse proxies (Nginx, Cloudflare) to avoid slash-mismatch 404s |
| `logErrors` | `boolean` | No | `true` | Same semantics as ergenecore's `logErrors` |

## Adapter Base Classes

Both adapters export the same base-class surface; only the import path differs (`@asenajs/ergenecore` vs `@asenajs/hono-adapter`):

| Export | Use |
|---|---|
| `type Context` | Handler/middleware context — always `import type`. Same `AsenaContext` API on both (`getParam` sync; `getQuery`/`getBody` async; `context.send(body, status?)`); `context.req` is a native `Request` on ergenecore, a `HonoRequest` on hono |
| `MiddlewareService` | Custom middleware base — implement `handle(context, next)` (its only abstract member) |
| `ValidationService` | Zod validator base — define `json()`, `query()`, `param()`, etc.; register with `@Middleware({ validator: true })` |
| `ConfigService` | `@Config()` base — override `globalMiddlewares()`, `onError()`, `onNotFound()`, `transport()` |
| `StaticServeService` | Static file serving with `@StaticServe({ root })`; hooks `rewriteRequestPath`, `onFound`, `onNotFound` |

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { MiddlewareService, type Context } from '@asenajs/ergenecore';

@Middleware()
export class AuthMiddleware extends MiddlewareService {
  async handle(context: Context, next: () => Promise<void>): Promise<any> {
    const token = context.headers['authorization']; // hono: context.req.header('authorization')
    if (!token) return context.send({ error: 'Unauthorized' }, 401);
    await next();
  }
}
```

## Built-in Middleware

Both adapters ship `CorsMiddleware` and `RateLimiterMiddleware`. **Always subclass with `@Middleware()` before use** — the classes live in `node_modules`, and Asena only scans your source folder, so the raw class is never discovered. Attach subclasses via `globalMiddlewares()`, `@Controller('/p', { middlewares: [...] })`, or `@Get({ path, middlewares: [...] })`.

### CorsMiddleware

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { CorsMiddleware } from '@asenajs/ergenecore'; // or '@asenajs/hono-adapter'

@Middleware()
export class RestrictedCors extends CorsMiddleware {
  constructor() {
    super({
      origin: ['https://example.com', 'https://app.example.com'], // or (origin) => boolean
      credentials: true,
      methods: ['GET', 'POST', 'PUT', 'DELETE'],
      allowedHeaders: ['Content-Type', 'Authorization'],
      exposedHeaders: ['X-Total-Count'],
      maxAge: 86400,
    });
  }
}
// super() with no args = { origin: '*' }
```

| Option | Type | Default |
|---|---|---|
| `origin` | `'*' \| string[] \| (origin: string) => boolean` | `'*'` |
| `credentials` | `boolean` | `false` |
| `methods` | `string[]` | `['GET','POST','PUT','PATCH','DELETE','OPTIONS']` |
| `allowedHeaders` | `string[]` | `['Content-Type', 'Authorization']` |
| `exposedHeaders` | `string[]` | `[]` |
| `maxAge` | `number` (seconds) | `86400` |

**Warning:** on ergenecore a bare origin string is not supported — wrap a single origin in an array.

### RateLimiterMiddleware

Token-bucket limiter; each middleware instance keeps its own bucket storage.

```typescript
import { RateLimiterMiddleware } from '@asenajs/ergenecore'; // or '@asenajs/hono-adapter'

@Middleware()
export class ApiRateLimiter extends RateLimiterMiddleware {
  constructor() {
    super({
      capacity: 100,
      refillRate: 100 / 60,                                        // tokens/second
      keyGenerator: (ctx) => ctx.getValue('user')?.id || 'anonymous',
      skip: (ctx) => ctx.getValue('user')?.role === 'admin',
      cost: (ctx) => ctx.req.url.includes('/search') ? 5 : 1,
      message: 'Rate limit exceeded. Please slow down.',
      statusCode: 429,
    });
  }
}
```

| Option | Type | Default |
|---|---|---|
| `capacity` | `number` | `100` |
| `refillRate` | `number` (tokens/sec) | `10` |
| `keyGenerator` | `(ctx) => string` | `x-forwarded-for` → `cf-connecting-ip` → `getRequestIp()` → `'unknown'` |
| `message` | `string` | `'Rate limit exceeded...'` |
| `statusCode` | `number` | `429` |
| `cost` | `number \| (ctx) => number` | `1` |
| `skip` | `(ctx) => boolean` | `undefined` |
| `cleanupInterval` | `number` (ms) | `60000` |
| `bucketTTL` | `number` (ms) | `600000` |

Sets `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, and `Retry-After` (on 429). Its sweep timer is released via an inherited `@OnStop` on `server.stop()` — no cleanup code needed in subclasses.

## Error & Not-Found Hooks

```typescript
import { Config } from '@asenajs/asena/decorators';
import { ConfigService, type Context } from '@asenajs/ergenecore'; // or '@asenajs/hono-adapter'
import { isHttpException, type NotFoundRequest } from '@asenajs/asena/adapter';

@Config()
export class ServerConfig extends ConfigService {
  globalMiddlewares() { return [GlobalCors, ApiRateLimiter]; }

  onError(error: Error, context: Context): Response | Promise<Response> {
    if (isHttpException(error)) {
      // error.message is the body already flattened to a string — don't re-wrap objects
      return error.getResponse?.() ?? context.send({ error: error.message }, error.status);
    }
    return context.send({ error: 'Internal Server Error' }, 500);
  }

  onNotFound(context: Context, request: NotFoundRequest): Response | Promise<Response> {
    return context.send({ title: 'Not Found', status: 404, instance: request.path }, 404);
  }
}
```

- `onError` = something your code **threw**; `onNotFound` = request matched **no route** (also fires for files `@StaticServe` cannot find). An unmatched route never reaches `onError`. A domain 404 (route exists, record doesn't) is a throw: `throw new HttpException(404, { error: 'User not found' })` from `@asenajs/asena/adapter`.
- **Every thrown error reaches `onError` first** — including `HttpException` and (on hono) hono's `HTTPException` and anything hono middleware raised. Fallback to the exception's own response only when no handler exists, it returns nothing, or it throws.
- Defaults with no hooks: unmatched route → `{"error":"Not Found"}` 404; unhandled error → `{"error":"Internal Server Error"}` 500 (thrown message goes to the log with stack, never to the caller).
- `request.path` is path-only (no origin/query), `request.method` is normalized — the same handler body works on both adapters.
- **Always match with `isHttpException()`, never `instanceof`.** On hono, apps throw core `HttpException` while `hono/basic-auth`, `hono/bearer-auth`, `hono/jwt` and hono's validator throw `HTTPException` — either `instanceof` misses half your 4xx and turns them into 500s. `isHttpException()` matches both and survives duplicate `@asenajs/asena` copies. Import `HTTPException` from `@asenajs/hono-adapter`, never from `hono/http-exception` (a second `hono` copy defeats even the guard). Ergenecore re-exports `HttpException` (same class object); hono-adapter deliberately does not.

## Hono-Specific Features

### @Override — native Hono middleware, no wrapper

Extend `AsenaMiddlewareService` (not the adapter's `MiddlewareService` — that binds `handle()` to the wrapped Context and a Hono `Context` parameter is a signature mismatch):

```typescript
import { Middleware, Override } from '@asenajs/asena/decorators';
import { AsenaMiddlewareService } from '@asenajs/asena/middleware';
import { compress } from 'hono/compress';
import type { Context as HonoContext, Next } from 'hono';

@Middleware()
export class CompressionMiddleware extends AsenaMiddlewareService {
  @Override()
  async handle(context: HonoContext, next: Next) {
    return compress()(context, next); // any hono ecosystem middleware works this way
  }
}
```

Use `@Override` only when you need Hono's native context; the wrapped Context is adapter-agnostic and preferred otherwise.

### Native request methods

On hono, `context.req.param('id')` / `context.req.json()` work alongside the unified `context.getParam('id')` / `context.getBody()`.

## Migrating Between Adapters / From Standalone Hono

**Between adapters:** change the factory call (tuple vs plain return) and the `Context` import path. Controllers, services, business logic stay unchanged.

**From standalone Hono:** routes move onto decorated controllers; handler bodies keep Hono's request API.

```typescript
// Before: app.get('/users/:id', (c) => c.json({ id: c.req.param('id') }));
@Controller('/users')
export class UserController {
  @Get({ path: '/:id' })
  async getUser(context: Context) {
    return context.send({ id: context.req.param('id') });
  }
}
```

Gains: IoC/`@Inject`, decorator routing, `ValidationService` (Zod), WebSocket integration, and `@Override` for existing hono middleware.

## Custom Adapters

Implement the `AsenaAdapter` interface from `@asenajs/asena/adapter`:

```typescript
import type { AsenaAdapter } from '@asenajs/asena/adapter';

export class MyCustomAdapter implements AsenaAdapter {
  async start(port: number): Promise<void> { /* ... */ }
  registerRoute(method: string, path: string, handler: Function): void { /* ... */ }
  // ... other required methods
}
```

**Warning:** `server.stop()` calls the adapter's `stop()` before any `@OnStop` hook; an adapter with WebSocket support must tear that layer down there (clear heartbeat timers, call the WebSocket transport's optional `destroy()`) — both official adapters do. Use the Ergenecore or hono-adapter source as the reference implementation.

Deeper detail: `https://asena.sh/raw/adapters/ergenecore.md`, `https://asena.sh/raw/adapters/hono.md`.
