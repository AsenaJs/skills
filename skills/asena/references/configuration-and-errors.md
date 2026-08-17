# AsenaJS Configuration & Error Handling

`@Config` server configuration (serve options, error/404 hooks, global middleware, transports) and the `HttpException` / `isHttpException()` error model — as of `@asenajs/asena` 0.10.x and adapters 3.x.

## Contents

- @Config Class
- serveOptions()
- onError()
- onNotFound()
- globalMiddlewares()
- transport()
- Config Bootstrap Facts
- HttpException
- ExceptionMapper Pattern
- Duplicate-Module Forensics
- Adapter Logging
- Deployment

## @Config Class

Exactly **one** `@Config` class per application (a second one throws at bootstrap — branch on `process.env` inside one class instead). Extend the adapter's `ConfigService`; every method is optional.

```typescript
import { Config } from '@asenajs/asena/decorators';
import { ConfigService, type Context } from '@asenajs/ergenecore';
import type { AsenaServeOptions } from '@asenajs/asena/adapter';

@Config()                    // or @Config('AppConfig')
export class AppConfig extends ConfigService {
  public serveOptions(): AsenaServeOptions {
    return {
      serveOptions: { development: true },
      wsOptions: { perMessageDeflate: true, idleTimeout: 120 },
    };
  }
}
```

**Note:** on hono-adapter the code is identical — import `ConfigService` and `Context` from `@asenajs/hono-adapter` instead.

Hook reference (all optional):

```typescript
serveOptions?(): AsenaServeOptions;
onError?(error: Error, context: C): Response | Promise<Response>;
onNotFound?(context: C, request: NotFoundRequest): Response | Promise<Response>;
globalMiddlewares?(): Promise<GlobalMiddlewareEntry[]> | GlobalMiddlewareEntry[];
transport?(): WebSocketTransport | AsenaTransportConfig | Promise<...>;
```

A `@Config` is a normal IoC component: `@Inject` works in it (logger, ExceptionMapper, a Redis service for `transport()`).

## serveOptions()

Returns `AsenaServeOptions = { serveOptions?: AsenaServerOptions; wsOptions?: WSOptions }`.

`AsenaServerOptions = Omit<ServeOptions, 'fetch' | 'routes' | 'websocket' | 'error'>` — those four are framework-managed and are compile errors here. Available: `hostname`, `reusePort`, `ipv6Only`, `tls`, `maxRequestBodySize`, `idleTimeout` (HTTP idle, Bun default **10 s**), `development`, `id`.

**Warning:** `port` inside `serveOptions` never takes effect — `AsenaServer.start()` overwrites it from the factory on every start. `hostname` is adapter-specific: hono honours `serveOptions.hostname`; ergenecore overwrites it with the factory value.

```typescript
const server = await AsenaServerFactory.create({ adapter, logger, port: 8080 });
const adapter = createErgenecoreAdapter({ hostname: '0.0.0.0' });   // ergenecore hostname
await server.start({ unix: '/tmp/asena.sock' });                    // unix socket is a start option, not a serve option
```

TLS goes under `serveOptions.tls` — `{ cert: Bun.file(...), key: Bun.file(...), ca?, passphrase?, serverName?, lowMemoryMode?, dhParamsFile? }`, or an array of such objects for SNI.

### wsOptions

| Option | Default | Notes |
|:--|:--|:--|
| `perMessageDeflate` | `false` | Must be present in the `wsOptions` object Bun receives (`boolean` or `{ compress?, decompress? }`) |
| `idleTimeout` | 120 s | WebSocket idle auto-close |
| `backpressureLimit` | 16 MB (Bun) | Triggers `drain` |
| `closeOnBackpressureLimit` | `false` | |
| `publishToSelf` | `false` | Receive own published messages |
| `sendPingStrategy` | `'adapter'` | `'adapter'` = framework `ws.ping()` heartbeat; `'native'` = Bun keep-alive |
| `heartbeatInterval` | — | ms, `'adapter'` strategy only; **no heartbeat is sent without it** |
| `maxPayloadLimit` | 16 MB | **Currently has no effect** — Bun's key is `maxPayloadLength`, so this is silently dropped; do not rely on it to bound message size |
| `sendPings` | — | **Ignored** — superseded by `sendPingStrategy` |

WebSocket options go in `wsOptions`, never `serveOptions.websocket` (compile error).

## onError()

Called for every unhandled error during request processing. Return a `Response` (or Promise) to own the answer.

```typescript
import { isHttpException } from '@asenajs/asena/adapter';

public onError(error: Error, context: Context) {
  if (isHttpException(error)) {
    // Let the exception answer with the body it carries
    return error.getResponse?.() ?? context.send({ error: error.message }, error.status);
  }

  console.error('[ERROR]', { message: error.message, stack: error.stack, url: context.req.url });
  return context.send({ error: 'Internal Server Error' }, 500);
}
```

**Always** branch on `isHttpException()` first — without it, every deliberate `401`/`403`/`404` (yours or an auth middleware's) reaches the client as a `500`. If `onError` is absent, returns nothing, or throws, the exception answers itself from its own status and body; anything that is not an `HttpException` becomes a 500. Never expose `error.stack` or internal messages in production responses.

## onNotFound()

Answers a request that matched **no route** — deliberately separate from `onError` (nothing threw).

```typescript
import type { NotFoundRequest } from '@asenajs/asena/adapter';

public onNotFound(context: Context, request: NotFoundRequest) {
  return context.send({ title: 'Not Found', status: 404, instance: request.path }, 404);
}
```

- `request.path` is path only (no origin/query); `request.method` is the upper-case verb; identical on both adapters.
- Default (no hook): both adapters answer `{"error":"Not Found"}`, status 404, `Content-Type: application/json`, plus one INFO log line `Route not found:` with `{ path, method, status }`.
- Global middlewares run **before** `onNotFound`, so a 404 still carries CORS headers.
- If the hook throws, the adapter logs it and falls back to the default 404 — it is **not** routed to `onError`.
- Domain 404s (route exists, record doesn't) are not this hook's job: `throw new HttpException(404, { code: 'USER_NOT_FOUND' })` — that reaches `onError`.
- Migration from 0.8: `NotFoundError` / `isNotFoundError` / `NOT_FOUND_ERROR` are removed — move that `onError` branch into `onNotFound`.

## globalMiddlewares()

**Warning:** it must be a **method**. A `middlewares = [...]` property on the config class is never read — the server starts normally, logs a startup warning, and the middleware silently never runs.

```typescript
public globalMiddlewares() {
  return [
    LoggerMiddleware,                                                    // all routes; array order = execution order
    { middleware: AuthMiddleware, routes: { include: ['/api/*', '/admin/*'] } },
    { middleware: RateLimitMiddleware, routes: { exclude: ['/health', '/metrics'] } },
    { middleware: JwtMiddleware, routes: { include: ['/api/*'], exclude: ['/api/public/*'] } },
  ];
}
```

```typescript
type GlobalMiddlewareEntry =
  | MiddlewareClass
  | { middleware: MiddlewareClass; routes?: { include?: string[]; exclude?: string[] } };
```

Pattern semantics:

- `*` matches any characters **including `/`** — `/api/*` covers `/api/v1/admin/users` (recursive, not single-segment). `**` compiles to the same regex as `*`.
- `/api/*` does **not** match the bare `/api` — list it explicitly if needed.
- Exact paths match literally; trailing slashes are normalized; `:param` matches a single segment.

The method may be `async` (e.g. build the list from an injected service).

## transport()

Two return forms:

1. **Bare `WebSocketTransport`** — cross-pod WebSocket only. Default when absent: `BunLocalTransport` (direct `server.publish()`, zero overhead single-pod).
2. **`AsenaTransportConfig` object** — WebSocket + microservice transports + interceptors:

```typescript
import { RedisTransport, RedisMicroserviceTransport } from '@asenajs/asena-redis';

@Config()
export class AppConfig extends ConfigService {
  @Inject('AppRedis')
  private redis: AppRedis;

  public transport() {
    return {
      websocket: new RedisTransport(this.redis),                        // optional; cross-pod WS
      microservice: new RedisMicroserviceTransport(
        { url: 'redis://localhost:6379' },
        { serviceName: 'order-service' },
      ),
      interceptors: [otelMessaging({ system: 'redis' })],               // optional
    };
  }
}
```

```typescript
interface AsenaTransportConfig {
  websocket?: WebSocketTransport;
  microservice?: MicroserviceTransport | Record<string, MicroserviceTransport>;  // single or named map
  interceptors?: MessagingInterceptor[];
}
```

A single microservice transport is registered under the name `default`; multi-broker projects pass a named map. See `https://asena.sh/raw/concepts/microservices.md`.

## Config Bootstrap Facts

- Config hooks are read during application setup, **after** every component's `@OnStart` has run (as of 0.10.x) — so a `@Config` building a transport from an injected service finds it already started, and a `@Config`'s own `@OnStart` can prepare what `transport()` / `globalMiddlewares()` uses.
- `serveOptions()` returning a Promise works at runtime (the adapters await it), but the interface declares a synchronous return — a class extending `ConfigService` cannot legally declare it `async`. Resolve async values before server start, or widen the type at the call site.
- Never hardcode secrets (TLS passphrases etc.) — use `process.env`.
- Environment config pattern: there is no config-file abstraction — read `process.env` directly (Bun auto-loads `.env` files) and branch on `NODE_ENV` inside `serveOptions()`/`onError()` (verbose errors in dev, generic 500 body + TLS from env in prod). Worked examples: `https://asena.sh/raw/guides/configuration.md`.

## HttpException

One class, in core — import from `@asenajs/asena/adapter`; the same import and throw work on ergenecore and hono-adapter:

```typescript
import { HttpException } from '@asenajs/asena/adapter';

throw new HttpException(404, 'Not Found');                                  // string body
throw new HttpException(400, { error: 'Invalid input', field: 'email' });   // JSON body (auto Content-Type)
throw new HttpException(429, 'Too Many Requests', { headers: { 'Retry-After': '60' } });
throw new HttpException(503, 'Service Unavailable', { statusText: 'Maintenance Mode' });
```

`new HttpException(status, body, options?)`:

| Parameter | Type | Required | Description |
|:--|:--|:--|:--|
| `status` | `HttpStatusCode \| number` | Yes | `ClientErrorStatusCode` / `ServerErrorStatusCode` enums (from `@asenajs/asena/web-types`) accepted |
| `body` | `string \| object` | No (default `''`) | Object bodies are serialized with `Content-Type: application/json` |
| `options` | `HttpExceptionInit` | No | Extends `ResponseInit` (headers, statusText) with `cause?: Error` |

### Match with `isHttpException()`, never `instanceof`

`instanceof` fails two ways: **two classes** (hono ecosystem throws `HTTPException`, your code throws `HttpException` — each `instanceof` misses the other) and **two module copies** (duplicate `hono` or `@asenajs/asena` → two distinct classes; see forensics below). `isHttpException()` matches both classes and — for core's `HttpException`, which brands each *instance* — survives duplicate copies. The guard narrows to `status`, optional `getResponse()`, and the usual `Error` members; it does **not** guarantee `body` (read `error.body` only on exceptions you threw yourself), hence the `?.` fallback pattern in `onError` above.

### Do not re-wrap `error.message` (double-encoding trap)

`message` is the body flattened to a **string** — for an object body, the serialized JSON:

```typescript
const error = new HttpException(404, { error: 'User not found' });
error.message        // '{"error":"User not found"}'  <- a string
error.getResponse()  // 404, body {"error":"User not found"}
```

So `context.send({ error: error.message }, error.status)` double-encodes an object body into `{"error":"{\"error\":\"User not found\"}"}`. Pick one style and keep it:

- **Exception owns the body**: throw any shape, handler answers `error.getResponse()`. Works for hono's `HTTPException` too (which has no `body`).
- **Handler owns the body**: throw **strings** only, handler builds the envelope from `error.message` + `error.status`.

### Hono-adapter import rule

Import `HTTPException` from `@asenajs/hono-adapter`, **never** from `hono/http-exception`. With two resolved copies of `hono`, the other copy's `HTTPException` is a different class that `isHttpException()` cannot recognise. `hono/basic-auth`, `hono/bearer-auth`, `hono/jwt` and hono's validator keep working — both classes are recognised and answered from their own status. `@asenajs/hono-adapter` deliberately does not re-export core's `HttpException` (one-case-different names would collide in autocomplete); import it from `@asenajs/asena/adapter`. `import { HttpException } from '@asenajs/ergenecore'` still works (same class object) but prefer the core path.

## ExceptionMapper Pattern

Centralized error mapping with DI — one place for all error types. `@Component` registers a plain injectable (no service semantics); `AuthError` stands for your app's own error class:

```typescript
// src/exceptions/ExceptionMapper.ts
import { Component } from '@asenajs/asena/decorators';
import { Scope } from '@asenajs/asena/decorators/ioc';
import type { Context } from '@asenajs/ergenecore';
import { isHttpException, isValidationError } from '@asenajs/asena/adapter';
import { ClientErrorStatusCode, ServerErrorStatusCode } from '@asenajs/asena/web-types';

@Component({ name: 'ExceptionMapper', scope: Scope.SINGLETON })
export class ExceptionMapper {
  public map(error: Error, context: Context): Response | Promise<Response> {
    // BEFORE isHttpException on purpose: a ValidationError IS an HTTP exception (status 400)
    if (isValidationError(error)) {
      const errors = error.issues.map((i) => ({ field: i.path.join('.'), message: i.message }));
      return context.send({ success: false, message: 'Validation error', errors }, ClientErrorStatusCode.BadRequest);
    }

    // Every deliberate HTTP status - yours and anything a hono middleware raised
    if (isHttpException(error)) {
      return error.getResponse?.() ?? context.send({ error: error.message }, error.status);
    }

    if (error instanceof AuthError) {   // own domain errors: instanceof is fine
      return context.send({ success: false, message: 'Authentication failed' }, ClientErrorStatusCode.Unauthorized);
    }

    console.error(`Internal Server Error: ${error.message}`, { path: context.req.path, stack: error.stack });
    return context.send({ success: false, message: 'Internal server error' }, ServerErrorStatusCode.InternalServerError);
  }
}
```

```typescript
// src/config/ServerConfig.ts
@Config()
export class ServerConfig extends ConfigService {
  @Inject('ExceptionMapper')
  private mapper: ExceptionMapper;

  public onError(error: Error, context: Context) {
    return this.mapper.map(error, context);
  }
}
```

Validation notes: match failed request validation with `isValidationError()`, not `instanceof ZodError` — the framework wraps it in an adapter-specific `ValidationError` (status 400) with adapter-agnostic `issues: { path, message, code }[]`. `error.cause` holds the original `ZodError` (Zod 4: the field is `issues`, `ZodError.errors` is removed). Unanswered, both adapters fall back to `{ "error": "Validation failed", "details": {...}, "target": "json" | "query" | "param" | "form" | "header" }`.

## Duplicate-Module Forensics

Two resolved copies of `hono` or `@asenajs/asena` produce two distinct exception classes (and two containers). Symptoms are **silent**: every deliberate `401`/`403` collapses to the generic `500` branch while the API keeps responding; DI and `@OnStart` misbehave as no-ops. Nothing looks broken until someone reads the status codes.

- As of adapters 3.0.0, `hono` and `zod` are **peer dependencies** the application owns — the adapter has no resolution slot of its own, so a project normally resolves exactly one copy (ergenecore is zero-dep; `zod` is a peer).
- Diagnose: `bun pm ls --all | grep hono` — a nested `@asenajs/hono-adapter/node_modules/hono` means a duplicate. The adapter also writes a startup warning when it resolves one.
- Repair: removing the duplicate from the lockfile is not enough — the stale `node_modules` must go too:

```bash
rm -rf node_modules bun.lock && bun install
```

## Adapter Logging

One rule, both adapters: **the framework's default log fires exactly when the framework's default response fires.**

| | Your hook answered | No hook, or it declined/threw |
|:--|:--|:--|
| Response | Yours | The framework's |
| Log | None — you own the record | `5xx` ERROR + stack · `4xx` DEBUG (INFO when the logger has no `debug`) · unmatched route `404` INFO |

Only `5xx` carries a stack (a wall of bot 401s cannot flood the error stream); unmatched routes log at INFO (`Route not found:` with `{ path, method, status }`). Pass `logErrors: false` to `createHonoAdapter` / `createErgenecoreAdapter` to silence all of it, including the 404 line. There is no setting that logs a request your own handler answered. Migration: ergenecore ≤1.5.x / hono-adapter ≤1.7.x logged only when no `onError` was registered — expect new error output whenever the framework answers.

## Deployment

```bash
asena build              # production build
bun dist/index.asena.js  # run it
```

- **Graceful shutdown is on by default**: `SIGTERM`/`SIGINT`/`SIGHUP` call `server.stop()` — new work stops, `@OnStop` hooks run while transports are still up, then everything releases in order.
- **Probes**: pass `health: { port }` to `AsenaServerFactory.create()`; point the restart policy at `{path}/live` and the load balancer at `{path}/ready` (default path `/healthz`). Readiness answers `503` for the whole drain, pulling the instance from rotation while it finishes work.
- Details: `https://asena.sh/raw/concepts/lifecycle.md`.
