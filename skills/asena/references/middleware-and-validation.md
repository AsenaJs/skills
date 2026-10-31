# Middleware & Validation

How to write Asena middleware, attach it at the four levels, use adapter built-ins, and validate requests with route context contracts or Zod validator classes (as of @asenajs/asena 0.12.x, ergenecore / hono-adapter 5.x).

## Contents

- Middleware class
- Attachment levels
- Stopping the chain
- Adapter built-ins must be subclassed
- Route context contracts
- Validators
- Schema factory caching (validatorCache)
- Validation error handling

## Middleware class

Extend the adapter's `MiddlewareService`, decorate with `@Middleware()`, implement `handle(context, next)`:

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { MiddlewareService, type Context } from '@asenajs/ergenecore';
// hono: import { MiddlewareService, type Context } from '@asenajs/hono-adapter';

@Middleware()
export class AuthMiddleware extends MiddlewareService {
  async handle(context: Context, next: () => Promise<void>): Promise<any> {
    const token = context.headers['authorization']?.replace('Bearer ', '');
    if (!token) {
      return context.send({ error: 'No token provided' }, 401);
    }
    try {
      const payload = await this.verifyToken(token);
      context.setValue('user', payload); // handlers read it via context.getValue('user')
      await next();
    } catch {
      return context.send({ error: 'Invalid token' }, 401);
    }
  }
}
```

Always `await next()` to pass control on — a bare `next()` without `await` is a bug. Middleware supports `@Inject` exactly like services.

Instead of returning a `Response` you may `throw new HttpException(401, 'Unauthorized')` (from `@asenajs/asena/adapter`) — same class on both adapters, routed to your `onError` handler.

## Attachment levels

Execution order, identical on both adapters: `globalMiddlewares()` (array order, pattern entries included) → controller `middlewares` → route `middlewares` → validation (contract or `validator`) → handler.

### 1. Global — a method on the @Config class

```typescript
import { Config } from '@asenajs/asena/decorators';
import { ConfigService } from '@asenajs/ergenecore';

@Config()
export class AppConfig extends ConfigService {
  public globalMiddlewares() {
    return [LoggerMiddleware, GlobalCors];
  }
}
```

**Warning:** `globalMiddlewares` MUST be a method. A `middlewares = [...]` property compiles but is never read by the framework — the middleware silently never runs.

### 2. Pattern-based — entries in the same array

```typescript
public globalMiddlewares() {
  return [
    LoggerMiddleware, // all routes
    { middleware: AuthMiddleware, routes: { include: ['/api/*', '/admin/*'] } },
    { middleware: ApiRateLimiter, routes: { exclude: ['/health', '/metrics'] } },
  ];
}
```

Pattern entries are not a separate stage — each runs at its array position, interleaved with unfiltered entries. Controller-level middleware always follows the whole global list.

### 3. Controller-level

```typescript
@Controller({ path: '/admin', middlewares: [AuthMiddleware, AdminRoleMiddleware] })
export class AdminController { /* applied to every route in the controller */ }
```

Controller middleware is attached to each route the controller declares — not registered as a `/admin/*` pattern. It does not run for a 404 under the prefix, nor for an `OPTIONS` preflight of a route without an `OPTIONS` handler (put `CorsMiddleware` in `globalMiddlewares()`), nor for another controller whose path merely starts with the same prefix. The options form needs `path` — `@Controller({ middlewares })` without it throws; use `path: '/'` for root.

### 4. Route-level

```typescript
@Post({ path: '/', middlewares: [AuthMiddleware] })
async create(context: Context) { /* ... */ }
```

## Stopping the chain

Three portable ways to short-circuit (handler never runs):

- `return context.send(...)` — any returned `Response` stops the chain
- `return false` — plain `403 Forbidden` on both adapters
- `throw new HttpException(status, message)` — routed to `onError`

**Warning:** Omitting `next()` is NOT portable. On hono, returning without calling `next()` ends the chain; on ergenecore, a `void`/`true` return without `next()` continues to the next middleware anyway. Always signal explicitly.

## Adapter built-ins must be subclassed

`CorsMiddleware`, `RateLimiterMiddleware`, and other adapter built-ins are **undecorated base classes**. Listing one directly in any middleware slot fails at startup with `… is not a component. Decorate it with @Middleware()` — component identity is not inherited. Subclass first:

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { CorsMiddleware, RateLimiterMiddleware } from '@asenajs/ergenecore';

@Middleware()
export class GlobalCors extends CorsMiddleware {
  constructor() {
    super({
      origin: ['https://example.com'], // or a predicate: (origin: string) => boolean
      credentials: true,
      methods: ['GET', 'POST', 'PUT', 'DELETE'],
      allowedHeaders: ['Content-Type', 'Authorization'],
      maxAge: 86400,
    });
  }
}

@Middleware()
export class ApiRateLimiter extends RateLimiterMiddleware {
  constructor() {
    super({
      capacity: 100,          // 100 requests
      refillRate: 100 / 60,   // per minute
      message: 'Rate limit exceeded',
      trustProxy: true,       // ONLY behind exactly one reverse proxy — omit when clients connect directly; see below
      keyGenerator: (ctx) => ctx.getValue('user')?.id || 'anonymous', // optional
      skip: (ctx) => ctx.getValue('user')?.role === 'admin',          // optional
      cost: (ctx) => (ctx.req.url.includes('/export') ? 10 : 1),      // optional
    });
  }
}
```

A disallowed CORS origin gets a normal response without CORS headers (the browser enforces the denial) — never use CORS as access control. With any `origin` other than `'*'`, the middleware sets `Vary: Origin`.

### Rate limiter client key (adapters 5.0+)

The default key is the **connecting peer** (`getRequestIp()`); `X-Forwarded-For` is ignored and `CF-Connecting-IP` is never read, so a client cannot mint a fresh bucket by forging a header.

| `trustProxy` | Client key |
|---|---|
| `false` (default) | The connecting peer |
| `true` | Rightmost `X-Forwarded-For` entry — one proxy layer, or a unix-socket listener |
| `string[]` | Walk `X-Forwarded-For` from the right; first address not in the list (IPv4 and IPv4-mapped forms both match) |
| `(address) => boolean` | Same walk with a predicate (CIDR checks) |

Both misconfigurations fail closed into **one shared bucket**: `true` behind two or more proxy layers (the outer proxy becomes the key — list every proxy instead), and a trusted peer that sends no usable `X-Forwarded-For` (logged once). Behind a proxy with `trustProxy` unset, every client shares the proxy's bucket. `maxBuckets` (default `10000`, positive integer) caps distinct clients in memory; the least recently seen is evicted and starts over full. For Cloudflare's header, use a `keyGenerator`.

### WebSocketOriginMiddleware

`CorsMiddleware` is not a WebSocket origin policy — a disallowed origin still upgrades, and the browser sends cookies with a cross-site handshake. For cookie-authenticated sockets, subclass `WebSocketOriginMiddleware` (both adapters) and list it **before** the auth middleware:

```typescript
import { Middleware, WebSocket } from '@asenajs/asena/decorators';
import { WebSocketOriginMiddleware } from '@asenajs/ergenecore'; // or '@asenajs/hono-adapter'

@Middleware()
export class ChatOrigin extends WebSocketOriginMiddleware {
  constructor() {
    super({ allowedOrigins: ['https://app.example.com'] }); // or (origin) => origin.endsWith('.example.com')
  }
}

@WebSocket({ path: 'chat', middlewares: [ChatOrigin, SessionAuth] })
export class ChatSocket extends AsenaWebSocketService<ChatData> { /* ... */ }
```

A non-allowed `Origin` gets **403**. `Origin` is compared exactly (scheme included, no trailing slash; no `'*'` value — use a predicate). A handshake without `Origin` is rejected unless `allowMissingOrigin: true` (non-browser clients). Don't put it on HTTP routes — it would reject every request without an `Origin`.

## Route context contracts

The recommended way to validate. `Context` from the adapter is a type **and** a value: `Context.define(schemaMap)` returns a context class to extend, and the handler takes it as its first parameter:

```typescript
import { Controller } from '@asenajs/asena/decorators';
import { Post } from '@asenajs/asena/decorators/http';
import { Context } from '@asenajs/ergenecore'; // hono: '@asenajs/hono-adapter' — value import, not `import type`
import { z } from 'zod';

export class CreateOrderContext extends Context.define({
  body: z.object({ quantity: z.coerce.number().int().positive() }),
  params: z.object({ shopId: z.coerce.number() }),
}) {
  get isBulk() {
    return this.input.body.quantity > 100; // getters are fine when read from the handler
  }
}

@Controller('/shops')
export class OrderController {
  @Post('/:shopId/orders')
  async create(context: CreateOrderContext) {
    const { shopId } = context.input.params;        // number
    const { quantity } = await context.getBody();   // validated body output, typed without a generic
    return context.send({ shopId, quantity, bulk: context.isBulk }, 201);
  }
}
```

| Key | Validates | Value handed to the schema |
|---|---|---|
| `body` | JSON body | Parsed JSON; `undefined` for an empty body |
| `form` | multipart / URL-encoded body | Object; repeated field → array |
| `query` | Query string | Object; repeated key → array, single stays a string |
| `params` | Route params | Object of strings |
| `headers` | Request headers | Object, lowercase names |

- Any **Standard Schema v1** library works (Zod is one); sync and async validation both supported. `define()` rejects unknown keys, non-Standard-Schema values, and `body` + `form` together; a misspelled key is a compile error.
- Order: `body`/`form` → `query` → `params` → `headers`; stops at the first failing target. Runs after the whole middleware chain on both adapters; a middleware that reads the body does not consume it.
- `context.input.<target>` holds the schema **output** (defaults, transforms, coercions applied); only declared targets exist on it. `getBody()` / `getParseBody()` return the validated `body` / `form` output; `getQuery()`, `getParam()`, `headers` still return **raw** values — read `context.input` for typed ones.
- `context.input` is per-request, frozen at top level (not deep-frozen), and **throws** when read before validation finished: in middleware, the constructor, or a field initializer.
- The handler gets a real instance of your subclass (`instanceof` and getters work). A class from one adapter's `define()` fails the boot on the other adapter.
- Errors: schema issues → **400**; malformed JSON → **400**; non-empty `body` whose content type is not `application/json`/`application/*+json` → **415**; `form` with another content type → **415**. Failures throw a `ValidationError` (`target` = contract key, `issues[].path` normalized to strings/numbers) that reaches `onError`; unhandled, the response is `{ "error": "Validation failed", "target": "body", "issues": [{ "message", "path" }] }`. An exception thrown *inside* a schema is not a validation failure — it reaches `onError` as thrown.
- A route cannot combine a contract with `validator`, and `validatorCache` on a contract route fails startup.
- **Requirements:** `experimentalDecorators` + `emitDecoratorMetadata`, and the class referenced as a **value** (same file, or imported without `import type`). Otherwise the route registers **without a contract** — nothing validated, `context.input` throws. Keep contracts next to their controller (e.g. `orders/CancelOrderContext.ts`).

Contracts check request *shape*; whether the request makes sense (cancelling a paid order) is the service's decision.

## Validators

Use a validator class instead of a contract when the route needs a hook, a schema built from injected services, or `response()` schemas for OpenAPI. Validators ARE middleware: `@Middleware({ validator: true })` on a class extending the adapter's `ValidationService`. Zod is a **peer dependency** (`bun add zod`); both adapters declare `zod: ^4.3.6` (they use the v4 top-level `flattenError` export).

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { ValidationService } from '@asenajs/ergenecore'; // hono: '@asenajs/hono-adapter'
import { z } from 'zod';

@Middleware({ validator: true })
export class CreateUserValidator extends ValidationService {
  json() {
    return z.object({
      name: z.string().min(3).max(50),
      email: z.string().email(),
      age: z.number().min(18).max(120),
    });
  }
}
```

Attach via the route option — the validator runs after the route's middleware and before the handler; on failure the handler is never called:

```typescript
@Post({ path: '/', validator: CreateUserValidator })
async create(context: Context) {
  const body = await context.getBody(); // schema OUTPUT: unknown keys stripped, defaults filled, coerce applied
  return context.send({ created: true, user: body }, 201);
}
```

**Typed body:** on a validated route, `getBody()` returns the schema's parsed output, not the raw payload — so `z.object()`'s unknown-key stripping genuinely blocks mass assignment. **Body only:** `query()`/`param()`/`header()` schemas validate and reject, but `getQuery()`/`getParam()`/`headers` still return raw request strings — a `z.coerce.number()` on a query param validates, then hands you the string; convert in the handler (or use a contract, whose `context.input.query` is typed output).

### Schema methods

Each returns a Zod schema or `{ schema, hook }`, and may be `async` (e.g. `@Inject` a repository for uniqueness `refine`s). They are schema **factories** — see `validatorCache` below for when they run:

| Method | Validates | Runtime |
|:-------|:----------|:--------|
| `json()` | JSON body | yes |
| `form()` | multipart / URL-encoded body | yes |
| `query()` | query string | yes |
| `param()` | route parameters | yes |
| `header()` | request headers | yes |
| `response()` | response shape by status code | no — OpenAPI documentation only |

**Warning:** a validator defining only `response()` fails the `@Middleware({ validator: true })` check at import time and the process exits — at least one runtime method is required.

### Hooks

```typescript
import { type ValidationSchemaWithHook, ValidationService } from '@asenajs/ergenecore';

json(): ValidationSchemaWithHook {
  return {
    schema: z.object({ email: z.string().email(), password: z.string().min(8) }),
    hook: (result, context) => {
      if (!result.success) return; // failure flows on to onError / default envelope
      context.setValue('validatedEmail', result.data.email);
    },
  };
}
```

Hook signature: `(result: z.ZodSafeParseResult<T>, context) => Response | void | Promise<Response | void>`. The data lives at `result.data` and only when `result.success` is true — branch on it first. Returning a `Response` answers the request as-is and `onError` never runs; returning a plain object does nothing — the hook cannot transform the payload, stash reshaped data with `setValue()` instead.

Adapter differences:

- Ergenecore runs the hook only on validation failure; hono on every attempt — always guard on `result.success`.
- Ergenecore's hook receives Asena's `Context` (`setValue`/`send`); hono's receives Hono's **native** context (`set()`/`get()`/`json()`). Both write the same per-request store, so handlers read either with `getValue()`.
- Order vs route middleware is the same on both: route middleware first, then the validator — state set by a route-level `AuthMiddleware` is already there. (hono-adapter below 5.0.0 validated before route middleware.)

## Schema factory caching (validatorCache)

`validatorCache` is a route option on every HTTP method decorator, both adapters:

```typescript
@Post({ path: '/', validator: CreateUserValidator, validatorCache: false })
```

| `validatorCache` | Factory (`json()`, `query()`, …) runs | Use when |
|---|---|---|
| `true` (default) | Once per route and target, at server start | The schema is fixed once the app is up |
| `false` | On every request that reaches validation | The factory builds its schema from settings that change after startup |

- Default mode awaits an `async` factory at startup; a factory that throws **fails the startup**, naming route and target. `@OnStart` hooks complete before routes initialize, so their state is visible either way.
- Only the prepared schema and hook are cached — validation and the hook still run per request, and a `refine` reading mutable state reads it on every validation. Each route of each server instance prepares its own.
- `validatorCache` on a route without a `validator`, or on a contract route, fails startup.

## Validation error handling

A failed validation always throws a `ValidationError` carrying HTTP status 400.

- **No `onError` handler** (or the handler returns nothing): for a validator class, both adapters answer with the flattened envelope `{ "error": "Validation failed", "details": { "formErrors": [], "fieldErrors": {...} }, "target": "json" }` — `target` names the failed part: `json` | `query` | `form` | `param` | `header`. A contract failure has no `details` and uses the contract key as `target` (`body` | `form` | `query` | `params` | `headers`) — see Route context contracts.
- **`onError` defined** on your `@Config` class: the `ValidationError` arrives there like any other error. Match with `isValidationError()`:

```typescript
import { Config } from '@asenajs/asena/decorators';
import { isValidationError, isHttpException } from '@asenajs/asena/adapter';
import { ConfigService, type Context } from '@asenajs/ergenecore';

@Config()
export class ServerConfig extends ConfigService {
  public onError(error: Error, context: Context): Response {
    if (isValidationError(error)) {
      // error.issues: [{ path, message, code }]; error.target; error.cause = original ZodError
      const errors = error.issues.map((i) => ({ field: i.path.join('.'), message: i.message }));
      return context.send({ success: false, message: 'Validation failed', errors }, 400);
    }
    if (isHttpException(error)) {
      return error.getResponse?.() ?? context.send({ error: error.message }, error.status);
    }
    return context.send({ success: false, message: 'Internal Server Error' }, 500);
  }
}
```

Rules:

- Check `isValidationError()` BEFORE `isHttpException()` — a `ValidationError` is itself an HTTP exception (status 400), so a generic branch placed above it swallows it.
- Match with the guards, never `instanceof` — the concrete class differs per adapter (ergenecore's extends `HttpException`, hono's extends hono's `HTTPException`).
- Zod 4 removed `ZodError.errors` — read `issues`, or `error.cause` for the full `ZodError` (e.g. `z.treeifyError(error.cause)`).
- A hook that returns a `Response` wins: it is sent as-is and `onError` is never reached for that request.
