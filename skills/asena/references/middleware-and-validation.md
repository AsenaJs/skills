# Middleware & Validation

How to write Asena middleware, attach it at the four levels, use adapter built-ins, and validate requests with Zod validators (as of @asenajs/asena 0.10.x, ergenecore / hono-adapter 3.x).

## Contents

- Middleware class
- Attachment levels
- Stopping the chain
- Adapter built-ins must be subclassed
- Validators
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

Execution order: `globalMiddlewares()` (array order, pattern entries included) → controller `middlewares` → route `middlewares` → handler.

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
      keyGenerator: (ctx) => ctx.getValue('user')?.id || 'anonymous', // optional
      skip: (ctx) => ctx.getValue('user')?.role === 'admin',          // optional
      cost: (ctx) => (ctx.req.url.includes('/export') ? 10 : 1),      // optional
    });
  }
}
```

**Note (3.1+):** A disallowed CORS origin now gets a normal response without CORS headers (browser enforces the denial) — earlier versions answered `403 CORS: Origin not allowed`; never use CORS as access control. With any `origin` other than `'*'`, the middleware sets `Vary: Origin`.

## Validators

Validators ARE middleware: `@Middleware({ validator: true })` on a class extending the adapter's `ValidationService`. Zod is a **peer dependency** (`bun add zod`); both adapters declare `zod: ^4.3.6` (they use the v4 top-level `flattenError` export).

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

Attach via the route option — the validator runs before the handler; on failure the handler is never called:

```typescript
@Post({ path: '/', validator: CreateUserValidator })
async create(context: Context) {
  const body = await context.getBody(); // schema OUTPUT: unknown keys stripped, defaults filled, coerce applied
  return context.send({ created: true, user: body }, 201);
}
```

**Typed body:** on a validated route, `getBody()` returns the schema's parsed output, not the raw payload (adapters 3.1+; earlier versions return the raw body regardless) — so `z.object()`'s unknown-key stripping genuinely blocks mass assignment. **Body only:** `query()`/`param()`/`header()` schemas validate and reject, but `getQuery()`/`getParam()`/`headers` still return raw request strings — a `z.coerce.number()` on a query param validates, then hands you the string; convert in the handler.

### Schema methods

Each returns a Zod schema or `{ schema, hook }`, and may be `async` (e.g. `@Inject` a repository for uniqueness `refine`s):

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
- Order vs route middleware: ergenecore validates **after** the route's middleware chain, hono registers validators **before** it — state set by a route-level `AuthMiddleware` is not there yet on hono. Global middleware always runs first on both.

## Validation error handling

A failed validation always throws a `ValidationError` carrying HTTP status 400.

- **No `onError` handler** (or the handler returns nothing): both adapters answer with the flattened envelope `{ "error": "Validation failed", "details": { "formErrors": [], "fieldErrors": {...} }, "target": "json" }` — `target` names the failed part: `json` | `query` | `form` | `param` | `header`.
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
