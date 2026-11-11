# AsenaJS Configuration & Error Handling

`@Config` server configuration (serve options, global middleware, transports) and error handling — `@ExceptionHandler` classes with `@Catch` / `@NotFound`, the `HttpException` family, and the deprecated `@Config.onError` / `onNotFound` hooks — as of `@asenajs/asena` 0.13.x.

## Contents

- @Config Class
- serveOptions()
- globalMiddlewares()
- transport()
- Config Bootstrap Facts
- Exception Handlers
- @Catch Resolution
- ExceptionContext and Scopes
- The Resolution Chain
- @NotFound()
- Deprecated: onError() and onNotFound()
- HttpException and Status Exceptions
- Validation Failures
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
globalMiddlewares?(): Promise<GlobalMiddlewareEntry[]> | GlobalMiddlewareEntry[];
transport?(): WebSocketTransport | AsenaTransportConfig | Promise<...>;
/** @deprecated Removed in 1.0 — use @Catch in an @ExceptionHandler class */
onError?(error: Error, context: C): Response | Promise<Response>;
/** @deprecated Removed in 1.0 — use @NotFound() in an @ExceptionHandler class */
onNotFound?(context: C, request: NotFoundRequest): Response | Promise<Response>;
```

A `@Config` is a normal IoC component: `@Inject` works in it (logger, a Redis service for `transport()`). Error handling does not belong here — it lives in `@ExceptionHandler` classes (below).

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
- Environment config pattern: there is no config-file abstraction — read `process.env` directly (Bun auto-loads `.env` files) and branch on `NODE_ENV` inside `serveOptions()` or an `@ExceptionHandler` (verbose errors in dev, generic 500 body + TLS from env in prod). Worked examples: `https://asena.sh/raw/guides/configuration.md`.

## Exception Handlers

Error handling lives in `@ExceptionHandler` classes. Their `@Catch(...types)` methods answer thrown errors, a `@NotFound()` method answers unmatched routes:

```typescript
// src/exceptions/ApiExceptionHandler.ts
import { ExceptionHandler } from '@asenajs/asena/decorators';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { Catch, NotFound, type ExceptionContext } from '@asenajs/asena/exception';
import { HttpException, isValidationError, type HttpExceptionLike, type NotFoundRequest } from '@asenajs/asena/adapter';
import { ICoreServiceNames } from '@asenajs/asena/ioc/types';
import type { ServerLogger } from '@asenajs/asena/logger';
import { type Context } from '@asenajs/ergenecore'; // on Hono: from '@asenajs/hono-adapter'

@ExceptionHandler()
export class ApiExceptionHandler {
  @Inject(ICoreServiceNames.SERVER_LOGGER)
  private logger!: ServerLogger;

  @Catch(HttpException)
  public http(error: HttpExceptionLike, ctx: ExceptionContext<Context>) {
    if (ctx.scope !== 'http') return;

    if (isValidationError(error)) {
      const errors = error.issues.map((i) => ({ field: i.path.join('.'), message: i.message }));
      return ctx.context.send({ error: 'Validation failed', errors }, 400);
    }

    // error.message is the body already flattened to a string — don't re-wrap objects
    return error.getResponse?.() ?? ctx.context.send({ error: error.message }, error.status);
  }

  @Catch()
  public fallback(error: unknown, ctx: ExceptionContext<Context>) {
    this.logger.error('Unhandled error', error);
    if (ctx.scope !== 'http') return;
    return ctx.context.send({ error: 'Internal Server Error' }, 500);
  }

  @NotFound()
  public missing(context: Context, request: NotFoundRequest) {
    return context.send({ error: 'Not Found', path: request.path }, 404);
  }
}
```

- `ExceptionHandler` comes from `@asenajs/asena/decorators`; `Catch`, `NotFound` and the `ExceptionContext` types from `@asenajs/asena/exception`.
- The class is an ordinary IoC component: found by the component scan (hand it to `imports` when it ships in a package), and `@Inject` works in it.
- Declaring one changes nothing until a `@Catch` matches — an error no `@Catch` matches is answered by the next link of the chain, exactly as with no handler class at all.
- **Several `@ExceptionHandler` classes may exist** — split by domain (`OrderExceptionHandler`, `PaymentExceptionHandler`); their methods merge into one resolution table.
- One `@Catch` may list several types: `@Catch(OrderNotFoundError, CustomerNotFoundError)`. Abstract base classes work: `@Catch(DomainError)` receives every subclass.
- `@Catch` and `@NotFound` methods inherited from a base class are registered; inherited `@Catch` methods are named in a boot log line. Two handler classes sharing a base with a `@Catch` therefore collide (duplicate type).
- A handler answers, it does not log for you: once a `@Catch` returns a `Response`, the adapter's default error log is skipped and the framework writes the original error at `debug` only (see Adapter Logging) — log in the handler, as `fallback` does.

Boot fails (before the port is bound) on:

- the same exception type in two `@Catch`es anywhere in the application — narrow one to a subclass;
- a second `@Catch()` catch-all, or a second `@NotFound()`, across all `@ExceptionHandler` classes;
- a `@Catch` argument that is not a class — an `undefined` here is almost always a circular import between the handler and the error's module (thrown at decoration time).

A class with no `@Catch` / `@NotFound` method logs a warning: registered, never called.

## @Catch Resolution

**Most specific wins**, independent of declaration order. For a thrown value:

1. **Identity** — the thrown value's prototype chain is walked upward; the first class with a `@Catch` answers. `@Catch(ContractValidationError)` beats `@Catch(HttpException)` beats `@Catch(Error)`. It is the `instanceof` test, so an error class built without `class … extends` (`util.inherits`, `Object.setPrototypeOf`) matches too.
2. **The `HttpException` brand** — `@Catch(HttpException)` also receives branded exceptions that are not core's class: hono's `HTTPException` (thrown by `hono/basic-auth`, `hono/jwt`, the hono adapter's `ValidationError`) and an `HttpException` from a duplicate copy of `@asenajs/asena`. Used only when step 1 found nothing narrower than `HttpException`.
3. **The catch-all** `@Catch()`.
4. Nothing matched — the next link of the chain answers (see The Resolution Chain).

`@Catch()` is not `@Catch(Error)`:

| Thrown value | `@Catch(Error)` | `@Catch()` |
|:--|:--|:--|
| An `Error` subclass no narrower `@Catch` claims — `ValidationError` and `HttpException` included | Answers | Answers only when no `@Catch(Error)` exists |
| `throw 'boom'`, `throw null`, `throw {}` | Never | Answers |

So a typed `@Catch` can read `error.message` safely — non-`Error` throws only ever reach the catch-all, which must type its parameter `unknown`.

**Warning:** `@Catch(Error)` — or the catch-all when there is none — receives `ValidationError`, `HttpException` and the auth 401s whenever no narrower `@Catch` claims them. A handler there that always answers `500` turns every validation `400` and every deliberate `401`/`403`/`404` into a `500`. Pair it with `@Catch(HttpException)` (as in the example above), or return nothing for `isHttpException(error)` so the exception answers itself. Without a `@Catch(HttpException)`, an info line at boot names the handler that will receive them.

## ExceptionContext and Scopes

A `@Catch` method receives `(error, ctx: ExceptionContext<Context>)`. `ExceptionContext` is a union discriminated by `ctx.scope` — narrow it before reading scope-specific fields:

| `ctx.scope` | Raised in | Fields | A returned value |
|:--|:--|:--|:--|
| `'http'` | Controller handlers, middleware, validators (contracts and validator classes) | `context` — the adapter `Context` (`send()`, `req`, headers) | A `Response` answers the request. Any other value is logged as an error and treated as a decline |
| `'microservice'`, `kind: 'message'` | `@MessagePattern` handlers | `message` (`MessageContext`), `data`, `source` (`OrderHandler.create`), `transport` (`default`) | Becomes the reply, delivered as a **success** (the caller's `send()` resolves with it, even `{ ok: false }` or `null`). A `Response` is not a reply: it is declined with a warning naming the `@Catch`. `undefined` and a `Response` rethrow the original error to the transport — the caller gets `REMOTE_ERROR`. An error the `@Catch` throws replaces the original one |
| `'microservice'`, `kind: 'event'` | `@EventPattern` handlers | same as `message` | **Ignored, with a warning.** The original error always reaches the transport, so its retry / DLQ policy applies unchanged; it waits for the `@Catch` for 5 s at most |
| `'event'` | `@On` handlers | `event` (`user.created`), `data`, `source` (absent for a handler registered imperatively on the emitter) | **Ignored, with a warning.** The default `[EventDispatchService]` error line is always logged |

Event scopes (`@EventPattern`, `@On`) are **observe-only**: a matching `@Catch` runs — page someone, count it, record it — but the failure continues exactly as if no `@Catch` existed. Returning a value there has no effect; the framework logs a warning naming the `@Catch` method.

One `@Catch` covers a failure wherever it surfaces:

```typescript
@Catch(DbConnectionError)
public database(error: DbConnectionError, ctx: ExceptionContext<Context>) {
  this.logger.error('Database unreachable', error); // runs in every scope
  if (ctx.scope !== 'http') return;                  // message: original error rethrown; events: observed only
  return ctx.context.send({ error: 'Service Unavailable' }, 503);
}
```

These three are the only scopes. A throwing `@Schedule` job keeps its own log-and-continue handling, and a throwing `@OnStart` still aborts the boot.

**No feedback loops.** A failure raised by work a running `@Catch` started — an event it emitted whose `@On` handler throws, a message it sent over `InMemoryTransport` — is not offered to `@Catch` again: the framework default handles it and a warning names the `@Catch` that started it. The guard follows the async call chain inside one process; a message that went through a real broker (Redis, Kafka) comes back as an ordinary failure, so a `@Catch` that publishes on every failure can still loop there.

## The Resolution Chain

```
Thrown error     →  @Catch (most specific)  →  @Config.onError (deprecated)    →  adapter default
Unmatched route  →  @NotFound()             →  @Config.onNotFound (deprecated) →  adapter's 404
```

- **Declining** = returning `undefined` (or nothing). The next link answers. On HTTP a non-`Response` return value also declines, with an error log; for a `@MessagePattern` a `Response` declines, with a warning. Declining is how a `@MessagePattern` caller gets to see the failure.
- **A handler that throws** on HTTP or in an event scope is logged (`@Catch handler ApiExceptionHandler.http() threw an error, falling back to the framework default`) and counts as a decline: the *original* error continues, never the handler's. A broken handler cannot make a request hang. The same holds for `@NotFound()`.
- **`@MessagePattern` is the exception:** an error the `@Catch` throws replaces the original failure and goes to the transport — throw there to send the caller a sanitized error instead of the internal one.
- **Adapter default for a thrown error:** any branded `HttpException` answers from its own status and body; a validation failure answers with the adapter's validation envelope (see Validation Failures); anything else becomes `{"error":"Internal Server Error"}`, status 500 — the message and stack go to the log, never to the caller.
- Outside HTTP the chain stops after `@Catch`: `@Config.onError` is HTTP-only.

## @NotFound()

Answers a request that matched **no route** — also a file `@StaticServe` cannot find. Signature: `(context, request: NotFoundRequest) => Response | void`.

- A routing 404 is **not a thrown error**: it never reaches `@Catch`, not even the catch-all.
- Exactly one `@NotFound()` per application.
- `request.path` is path only (no origin/query); `request.method` is the upper-case verb; identical on both adapters.
- Default (nothing answered): both adapters answer `{"error":"Not Found"}`, status 404, `Content-Type: application/json`, plus one INFO log line `Route not found:` with `{ path, method, status }`.
- Global middlewares run **before** it, so a 404 still carries CORS headers.
- A domain 404 (route exists, record doesn't) is a throw, not this hook's job: `throw new NotFoundException({ code: 'USER_NOT_FOUND' })` — `@Catch(NotFoundException)` or `@Catch(HttpException)` answers it, or it answers itself.

## Deprecated: onError() and onNotFound()

`@Config.onError(error, context)` and `@Config.onNotFound(context, request)` are **deprecated and removed in 1.0**. Each logs a deprecation warning at boot. They remain the last link before the adapter default: `onError` runs only after no `@Catch` answered, `onNotFound` only after `@NotFound()` declined (or does not exist). Both are HTTP-only — in headless mode they are ignored with a warning.

Never write new code against them. Use `@Catch` / `@NotFound()` in an `@ExceptionHandler` instead; when a project still has them, move their branches into `@Catch` methods.

## HttpException and Status Exceptions

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

### Ready-made status exceptions

One `HttpException` subclass per common status, also from `@asenajs/asena/adapter`. Each takes `(body?, options?)` — the status is fixed and `body` defaults to the reason phrase:

```typescript
import { ConflictException, NotFoundException } from '@asenajs/asena/adapter';

throw new NotFoundException();                                       // 404, body 'Not Found'
throw new ConflictException({ error: 'Email already registered' });  // 409, JSON body
```

| Status | Class | Status | Class |
|:--|:--|:--|:--|
| 400 | `BadRequestException` | 413 | `PayloadTooLargeException` |
| 401 | `UnauthorizedException` | 415 | `UnsupportedMediaTypeException` |
| 403 | `ForbiddenException` | 418 | `ImATeapotException` |
| 404 | `NotFoundException` | 422 | `UnprocessableEntityException` |
| 405 | `MethodNotAllowedException` | 500 | `InternalServerErrorException` |
| 406 | `NotAcceptableException` | 501 | `NotImplementedException` |
| 408 | `RequestTimeoutException` | 502 | `BadGatewayException` |
| 409 | `ConflictException` | 503 | `ServiceUnavailableException` |
| 410 | `GoneException` | 504 | `GatewayTimeoutException` |
| 412 | `PreconditionFailedException` | 505 | `HttpVersionNotSupportedException` |

Prefer them over `new HttpException(404, …)`: a status is a runtime number, a class is something `@Catch(NotFoundException)` can single out. Any status without a class (e.g. 429) stays `new HttpException(429, …)`.

### Custom domain errors

Domain errors are plain classes; `@Catch` matches them by prototype chain, so an abstract base catches the whole family:

```typescript
export abstract class DomainError extends Error {}
export class InsufficientStockError extends DomainError {}
export class OrderAlreadyShippedError extends DomainError {}

@Catch(DomainError)
public domain(error: DomainError, ctx: ExceptionContext<Context>) {
  if (ctx.scope !== 'http') return;
  return ctx.context.send({ error: error.name, message: error.message }, 409);
}
```

An error that should always map to one status can extend a status exception instead (`class InsufficientStockError extends ConflictException {}`) — it then answers itself with no handler at all, and `@Catch(ConflictException)` / `@Catch(HttpException)` still receive it.

### Match with `isHttpException()`, never `instanceof`

Inside a handler, test HTTP exceptions with `isHttpException()`, never `instanceof`. `instanceof` fails two ways: **two classes** (hono ecosystem throws `HTTPException`, your code throws `HttpException` — each `instanceof` misses the other) and **two module copies** (duplicate `hono` or `@asenajs/asena` → two distinct classes; see forensics below). `isHttpException()` matches both classes and — for core's `HttpException`, which brands each *instance* — survives duplicate copies; `@Catch(HttpException)` uses the same brand. The guard narrows to `HttpExceptionLike`: `status`, optional `getResponse()`, and the usual `Error` members; it does **not** guarantee `body` (read `error.body` only on exceptions you threw yourself), hence the `?.` fallback pattern in the handler above. Your own domain errors are a single class you own — `instanceof` is fine for them.

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

## Validation Failures

Every failed request validation throws a status-400 `HttpException` subclass that `isValidationError()` recognises. Which class depends on the validation style:

| Validation style | Class thrown | Import it from |
|:--|:--|:--|
| Route context contract (`Context.define()`) | `ContractValidationError` | `@asenajs/asena/adapter` |
| Validator class (`ValidationService`) | The adapter's `ValidationError` | `@asenajs/ergenecore` / `@asenajs/hono-adapter` |

- `@Catch(HttpException)` receives both, on both adapters (hono's `ValidationError` extends hono's `HTTPException` and is matched through the brand). Check `isValidationError(error)` **first** inside it, as the example handler does — a generic HTTP branch placed above it swallows the validation envelope.
- To give validation its own handler, catch both classes: `@Catch(ContractValidationError, ValidationError)`. Most-specific-wins then routes them there ahead of `@Catch(HttpException)`.
- Never `instanceof ZodError` — the failure is always wrapped. `error.issues` is adapter-agnostic: `{ path: (string | number)[], message, code? }[]`; `error.target` names the failed part; `error.cause` holds the validator's own error (the `ZodError` for a validator class — Zod 4: `issues`, `ZodError.errors` is removed).
- Declined (or nothing matched): a validator class answers `{ "error": "Validation failed", "details": {...}, "target": "json" | "query" | "param" | "form" | "header" }`; a contract answers `{ "error": "Validation failed", "target": "body", "issues": [{ "message", "path" }] }`.
- A validator hook that returns a `Response` answers as-is — no `@Catch` runs for that request.

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

| | Your handler answered (`@Catch`, `@NotFound()`, or a deprecated config hook) | Nothing answered: no handler, or every one declined/threw |
|:--|:--|:--|
| Response | Yours | The framework's |
| Log | You own the record. For a `@Catch` the framework adds one DEBUG line with the original error (`@Catch handler ApiExceptionHandler.http() answered:`) | `5xx` ERROR + stack · `4xx` DEBUG (INFO when the logger has no `debug`) · unmatched route `404` INFO |

Only `5xx` carries a stack (a wall of bot 401s cannot flood the error stream); unmatched routes log at INFO (`Route not found:` with `{ path, method, status }`). Pass `logErrors: false` to `createHonoAdapter` / `createErgenecoreAdapter` to silence all of it, including the 404 line. The DEBUG line is the only trace of an error a `@Catch` answered (a `@MessagePattern` reply included), and a logger without `debug` writes nothing — log inside the `@Catch` when the error matters.

## Deployment

```bash
asena build              # production build
bun dist/index.asena.js  # run it
```

- **Graceful shutdown is on by default**: `SIGTERM`/`SIGINT`/`SIGHUP` call `server.stop()` — new work stops, `@OnStop` hooks run while transports are still up, then everything releases in order.- **Probes**: pass `health: { port }` to `AsenaServerFactory.create()`; point the restart policy at `{path}/live` and the load balancer at `{path}/ready` (default path `/healthz`). Readiness answers `503` for the whole drain, pulling the instance from rotation while it finishes work.
- Details: `https://asena.sh/raw/concepts/lifecycle.md`.
