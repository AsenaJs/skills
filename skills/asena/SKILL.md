---
name: asena
description: Expert guidance for AsenaJS, the Bun IoC web framework with Spring Boot-style decorators and field-based DI. Use when creating or modifying AsenaJS projects or code importing @asenajs packages - controllers, routing, services, middleware, validation (route context contracts, Zod), error handling (exception handlers), authentication and sessions, WebSocket namespaces, Redis Streams/Kafka microservices, Redis caching, events, cron tasks, static files, Drizzle repositories, OpenAPI, OpenTelemetry, testing with mockComponent/createTestApp/createWebTest, the asena CLI. Triggers - asena, asenajs, Context.define, AsenaServerFactory, createErgenecoreAdapter, createHonoAdapter, ergenecore, hono-adapter, asena-auth, @Controller, @Service, @Inject, @Value, @OnStart, @ExceptionHandler, @Catch, @NotFound, HttpException, @Protected, @Auth, @BetterAuth, AuthGuard, authComponents, requireAuthSession, @WebSocket, @MessagePattern, @StaticServe, @FrontendController, AsenaLogger, ulak, asena-config, asena doctor, asena console.
version: 0.5.0
license: MIT
---

# AsenaJS

AsenaJS is a decorator-based IoC web framework that runs ONLY on Bun. It looks like NestJS or Spring Boot but is neither: decorator names, import paths, adapter factories, and lifecycle hooks all differ. Never write AsenaJS code from memory of other frameworks — follow this file for the core rules and read the matching file in `references/` (routing table at the bottom) before writing code in any feature area.

## Version matrix

Package versions are NOT aligned — do not assume one version number across packages. As of this skill's sync (the `@asenajs/asena` 0.13.0 release). If an installed package resolves lower than the table shows, the behaviour this file describes for it may not apply — check before relying on it.

| Package | npm | Version | Peer deps |
|---|---|---|---|
| Core framework | `@asenajs/asena` | 0.13.0 | — (`reflect-metadata` only runtime dep) |
| CLI | `@asenajs/asena-cli` | 0.13.0 | — (no `@asenajs/asena` dependency; reads the project's own core) |
| Hono adapter | `@asenajs/hono-adapter` | 5.2.0 | `hono` `>=4.12.9`, `zod` `^4.3.6`, core `^0.12.0 \|\| ^0.13.0` |
| Ergenecore adapter (native Bun) | `@asenajs/ergenecore` | 5.1.0 | `zod` `^4.3.6`, core `^0.12.0 \|\| ^0.13.0` |
| Auth (sessions, Better Auth, route guards) | `@asenajs/asena-auth` | 1.0.1 | core `^0.12.2 \|\| ^0.13.0`; `better-auth` `>=1.2.0 <2` optional (Better Auth provider only) |
| Redis (cache + transports) | `@asenajs/asena-redis` | 4.1.0 | `redis` `^5.12.1`, core `^0.11.0 \|\| ^0.12.0 \|\| ^0.13.0` |
| Kafka (client + transport) | `@asenajs/asena-kafka` | 4.0.2 | `kafkajs` `^2.2.4`, core `^0.11.0 \|\| ^0.12.0 \|\| ^0.13.0` |
| OpenAPI | `@asenajs/asena-openapi` | 3.2.1 | `zod` `^4.3.6`, core `^0.12.0 \|\| ^0.13.0` |
| OpenTelemetry | `@asenajs/asena-otel` | 3.0.2 | `@opentelemetry/*` SDK family (`api` `^1.9.0`, SDK packages `^2.6.0`), core `^0.11.0 \|\| ^0.12.0 \|\| ^0.13.0` |
| Drizzle ORM | `@asenajs/asena-drizzle` | 4.1.1 | `drizzle-orm` `^0.45.2` (+ `pg` `>=8` or `mysql2` `>=2` per driver), core `^0.11.0 \|\| ^0.12.0 \|\| ^0.13.0` |
| Logger | `@asenajs/asena-logger` | 3.0.2 | core `^0.11.0 \|\| ^0.12.0 \|\| ^0.13.0` |

Both adapters and `asena-openapi` peer `@asenajs/asena` `^0.12.0 || ^0.13.0`; `asena-auth` peers `^0.12.2 || ^0.13.0`, is adapter-agnostic, and ships a `@asenajs/asena-auth/testing` subpath — on core 0.12 it needs ergenecore 5.0.2+ or hono-adapter 5.0.3+ (cookie deletion with attributes, `Set-Cookie` kept on error responses), on core 0.13 the adapter versions in the table; the other add-ons (redis, kafka, otel, drizzle, logger) peer `^0.11.0 || ^0.12.0 || ^0.13.0`. On core 0.13 every `@asenajs/*` package that peers core must be at least the version in the table — earlier releases do not accept 0.13 as a peer. `reflect-metadata` comes in through core. `asena-cli` 0.11.0 depends on `@asenajs/asena` `^0.11.0` (0.11.1 and later have no core dependency): on a 0.12 or 0.13 project it nests a second copy at `node_modules/@asenajs/asena-cli/node_modules/@asenajs/asena` that app code never resolves but `asena doctor`'s `duplicate-packages` check reports — upgrade the CLI to remove it. Every package requires **Bun >= 1.4.0**. There is no Node.js support — both adapters bind to `Bun.serve()`.

Error handling follows `@asenajs/asena` **0.13.0**: `@ExceptionHandler` / `@Catch` / `@NotFound`, the `@asenajs/asena/exception` subpath and the ready-made status exceptions (`NotFoundException`, …) need core 0.13.0; `@Config.onError` / `onNotFound` are deprecated in 0.13.0 and removed in 1.0.

Run `asena doctor` in an existing project to see the installed versions and whether any peer range is unsatisfied.

## Project setup

Fastest path: `bun install -g @asenajs/asena-cli && asena create` (interactive; or `asena create my-app --adapter ergenecore --logger`).

Manual install — the adapter's peers are the APP's dependencies; always install them explicitly:

```bash
# Ergenecore (default choice for new projects)
bun add @asenajs/asena @asenajs/ergenecore @asenajs/asena-logger zod
bun add -D @asenajs/asena-cli

# Hono adapter (when reusing the Hono middleware ecosystem)
bun add @asenajs/asena @asenajs/hono-adapter @asenajs/asena-logger hono zod
bun add -D @asenajs/asena-cli
```

**Always** enable both decorator flags in `tsconfig.json` — without them components silently fail to register:

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

Then `asena init` to create `asena-config.ts` (build + source-folder config).

## Bootstrap

The two adapter factories are asymmetric — this is the most common hallucination target:

- `createErgenecoreAdapter(options?)` returns **the adapter alone**; logger optional.
- `createHonoAdapter({ logger })` returns **a tuple** `[adapter, logger]` and requires a logger.

```typescript
// src/index.ts — Ergenecore
import { AsenaServerFactory } from '@asenajs/asena';
import { createErgenecoreAdapter } from '@asenajs/ergenecore';
import { logger } from './logger';

const adapter = createErgenecoreAdapter();

const server = await AsenaServerFactory.create({ adapter, logger, port: 3000 });

await server.start();
```

```typescript
// src/index.ts — Hono adapter (only the adapter lines differ)
import { createHonoAdapter } from '@asenajs/hono-adapter';

const [adapter] = createHonoAdapter({ logger });
```

```typescript
// src/logger.ts
import { AsenaLogger } from '@asenajs/asena-logger';

export const logger = new AsenaLogger();
```

Rules:

- Components declared in the entry file must sit **above** the `AsenaServerFactory.create()` call — anything below is not evaluated when components are collected (Asena logs a warning naming it).
- Dev: `bun run --hot src/index.ts`. Prod: `asena build && bun dist/index.asena.js`.
- **Never** use `asena dev start` in production — it is for quick local testing and slated for removal.

## Import cheatsheet

All **core** decorators come from `@asenajs/asena` subpaths. Only `Context` (type, and the `Context.define()` value), the middleware/validator/config base classes, and adapter built-ins come from the adapter package (`@asenajs/ergenecore` or `@asenajs/hono-adapter`). Add-on packages export their own decorators: `@Database`/`@Repository`/`@Drizzle`/`@Transaction` (`@asenajs/asena-drizzle`), `@Redis`/`@RedisCache` (`@asenajs/asena-redis`), `@Kafka` (`@asenajs/asena-kafka`), `@Otel` (`@asenajs/asena-otel`), `@Hidden` (`@asenajs/asena-openapi`), `@Auth`/`@BetterAuth` (`@asenajs/asena-auth`).

| Import from | Symbols |
|---|---|
| `@asenajs/asena` | `AsenaServerFactory`, `AsenaServer`, `LifecycleState` |
| `@asenajs/asena/decorators` | `Controller`, `Service`, `Middleware`, `Config`, `ExceptionHandler`, `WebSocket`, `Schedule`, `EventService`, `MessageController`, `FrontendController`, `PostProcessor`, `StaticServe`, `Override`, `Component` (also re-exported here) |
| `@asenajs/asena/decorators/http` | `Get`, `Post`, `Put`, `Patch`, `Delete`, `Head`, `Options`, `All`, `Route`, `Page` |
| `@asenajs/asena/decorators/ioc` | `Inject`, `Value`, `Strategy`, `Implements`, `OnStart`, `OnStop`, `Component`, `Scope` |
| `@asenajs/asena/exception` | `Catch`, `NotFound`, `ExceptionContext` (type) and its scope types |
| `@asenajs/asena/adapter` | `HttpException`, status exceptions (`BadRequestException`, `UnauthorizedException`, `ForbiddenException`, `NotFoundException`, `ConflictException`, …), `ContractValidationError`, `isHttpException`, `isValidationError`, `HttpExceptionLike`, `NotFoundRequest`, adapter SPI types |
| `@asenajs/asena/auth` | Auth **contract** only: `AsenaAuthProvider`, `AuthUser`, `AuthSession`, `Protected`, `Public`, `Roles`, `getAuthSession`, `requireAuthSession` (core never enforces the markers — `@asenajs/asena-auth` does; see pitfalls and `references/auth.md`) |
| `@asenajs/asena/web-socket` | `AsenaWebSocketService`, `Socket` (type; alias of `AsenaSocket`) |
| `@asenajs/asena/microservice` | `MessagePattern`, `EventPattern`, `InMemoryTransport`, transport SPI |
| `@asenajs/asena/messaging` | `ulak`, `Ulak`, `UlakError`, `UlakErrorCode` |
| `@asenajs/asena/event` | `On`, `emitter`, `EventEmitter` |
| `@asenajs/asena/schedule` | `CronRunner`, `AsenaSchedule` (type) |
| `@asenajs/asena/test` | `mockComponent`, `mockComponentAsync`, `createTestApp`, `createWebTest`, `silentLogger`, `createTestUlakStub`, `createMockFromClass`, `createDeepMock` |
| `@asenajs/asena/http-status` | `HttpStatus` |
| `@asenajs/asena/web-types` | `HttpMethod`, status-code enums (`ClientErrorStatusCode`, `ServerErrorStatusCode`, …) |
| `@asenajs/asena/middleware` | `AsenaMiddlewareService`, `AsenaValidationService` (framework base abstractions — app code normally extends the ADAPTER's classes instead) |
| `@asenajs/asena/logger` | `ServerLogger` (interface) |
| `@asenajs/asena/ioc/types` | `ICoreServiceNames`, `ComponentPostProcessor`, decorator param types |
| adapter package | `Context` (type and value — `Context.define()`), `MiddlewareService`, `ValidationService`, `ConfigService`, `ValidationError` (the validator-class failure), built-in middlewares (`CorsMiddleware`, `RateLimiterMiddleware`, `WebSocketOriginMiddleware`, …) |

**Never** import `Controller`/`Get`/`Inject` from the adapter package, and never import `Context` from the core package.

## Minimal controller + service

```typescript
import { Controller, Service } from '@asenajs/asena/decorators';
import { Get, Post } from '@asenajs/asena/decorators/http';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { type Context } from '@asenajs/ergenecore'; // on Hono: from '@asenajs/hono-adapter'

@Service()
export class UserService {
  private users = new Map<string, { id: string; name: string }>();

  async getById(id: string) {
    return this.users.get(id) ?? null;
  }
}

@Controller('/users')
export class UserController {
  @Inject(UserService)
  private userService!: UserService;

  @Get('/:id')
  async getUser(context: Context) {
    const id = context.getParam('id');          // getParam is sync
    const user = await this.userService.getById(id);
    return context.send(user);                  // send(data, status?) — status defaults to 200
  }

  @Post('/')
  async createUser(context: Context) {
    const body = await context.getBody();       // getBody/getQuery are async — await them
    return context.send({ created: true, data: body }, 201);
  }
}
```

## Core rules

- **Always** inject with `@Inject` on fields. **Never** use constructor injection — injected values are undefined inside constructors.
- **Always** do post-injection setup in a `@OnStart` method. `@PostConstruct` is a deprecated alias since 0.9 (renamed to `@OnStart`), and its timing changed in 0.10 (runs from `server.start()`, not during container scan).
- **Never** assign to an injected field — injected fields are read-only accessors and assignment throws. In tests use `overrides`/`mockComponent` instead.
- `@Inject` resolves registered components only — there is no value/token registry for arbitrary strings. Use **`@Value('KEY', { parse?, default? })`** for environment configuration instead of reading `process.env` inside the class. Exception: framework factory tokens (`ulak('/chat')`, `ulak.messages('svc')`, `emitter()`) are valid `@Inject` arguments — they resolve framework-provided components.
- `@Value` fails the component's construction when the variable is unset and no `default` was given. `default`'s presence is what counts, so `0`/`''`/`null` are honoured; `parse` runs on the raw env string only. Unlike `@Inject` fields, a `@Value` field is plain and writable.
- Components that ship in a **package** are never scanned (`node_modules` is not walked) — hand them to `AsenaServerFactory.create({ imports: [...] })`. `imports` adds to whatever else was found; every entry must carry its own component decorator or it throws.
- `@Strategy` always yields an **array**: zero implementations gives `[]`, not an error. Guard for empty.
- **Always** `await` every promise in handlers and lifecycle methods — an unawaited rejected promise shuts the whole server down.
- **Never** enable `minify.identifiers` — component registration is name-based and mangled class names break it in production only. `keepNames: true` does NOT rescue it (Bun 1.4.0's bundler drops class names under identifier minification regardless); `identifiers: false` is the only rule, and `asena build` now forces it. Registering by explicit string name (`@Service('UserService')`) is good practice anyway.
- Use `server.resolve()` only outside components and only after `server.start()`; inside components always use `@Inject`.

## Architecture discipline

Asena promotes a strict layered architecture: `Request → Controller → Service → Repository → Database`. Enforce these boundaries in every file you generate or modify — violations here are the most common quality failure in generated code:

- **Controllers stay thin** — parse the request, call a service, shape the response. **Never** put business logic, direct repository/database access, or multi-service orchestration in a controller.
- **A repository belongs to exactly one domain service.** Inject `OrderRepository` only into `OrderService`. **Never** inject a repository into another domain's service or into a controller — that bypasses the owning domain's rules and couples the caller to the schema.
- **Cross-domain access goes through the owning service.** When `InvoiceService` needs order data, inject `OrderService`, not `OrderRepository`. If two services would end up needing each other, extract the shared logic into a third service, or decouple the reaction with an in-process event (`@On`).
- **Services own the business invariants.** Stock checks, state transitions, ownership rules live in the service — not in the controller and not in the repository. Route context contracts and validators check request *shape*; services check *meaning*.
- **Write object-oriented code.** Model a domain as a service with intention-revealing methods (`reserveStock()`, `cancelOrder()`), not a data bag mutated from outside. Keep layer types where they belong: only controllers touch `Context`; repositories return domain data, never HTTP concerns.
- **Match the project's existing layout.** The CLI scaffolds type-based folders (`src/controllers/`, `src/services/`, …); follow whatever structure the project already uses instead of inventing a new one. If the existing code deliberately violates a boundary above, flag it to the user — don't silently rewrite it, and don't copy the violation into new code.

## Middleware and validation

Middleware is a class extending the **adapter's** `MiddlewareService`:

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { MiddlewareService, type Context } from '@asenajs/ergenecore';

@Middleware()
export class AuthMiddleware extends MiddlewareService {
  async handle(context: Context, next: () => Promise<void>): Promise<any> {
    // reject: throw an HttpException (e.g. UnauthorizedException). pass: await next()
    await next();
  }
}
```

Four attachment levels: global (returned from `globalMiddlewares()` in the `@Config` class), pattern-based include/exclude, controller-level (`@Controller({ path, middlewares: [...] })`), route-level (`@Get({ path, middlewares: [...] })`).

- `globalMiddlewares()` **must be a method** on the `@Config` class. As a property it is silently ignored.
- Controller-level middleware runs only on the routes that controller declares — not on 404s under its prefix and not on `OPTIONS` preflight. Put `CorsMiddleware` in `globalMiddlewares()`.
- Adapter built-ins (`CorsMiddleware`, `RateLimiterMiddleware`, …) must be subclassed with `@Middleware()` before use — never list them directly: `@Middleware() export class GlobalCors extends CorsMiddleware {}`.

Per-route pipeline, identical on both adapters: global → controller → route middleware → validation → handler. On a protected route the `@asenajs/asena-auth` guard runs between controller and route middleware.

**Validate with a route context contract** (recommended) — a context subclass built with the adapter's `Context.define()`, taken as the handler's first parameter. The handler reads typed, validated values from `context.input`:

```typescript
import { Controller } from '@asenajs/asena/decorators';
import { Post } from '@asenajs/asena/decorators/http';
import { Context } from '@asenajs/ergenecore'; // a VALUE import — never `import type` here
import { z } from 'zod';

export class CreateUserContext extends Context.define({
  body: z.object({ name: z.string().min(1), email: z.email() }),
  query: z.object({ notify: z.stringbool().default(false) }),
}) {}

@Controller('/users')
export class UserController {
  @Post('/')
  async create(context: CreateUserContext) {
    const { name, email } = context.input.body; // typed schema output
    return context.send({ name, email, notify: context.input.query.notify }, 201);
  }
}
```

Targets: `body` | `form` | `query` | `params` | `headers`; any Standard Schema v1 library works. A contract class imported with `import type` (or a missing `emitDecoratorMetadata`) registers the route with **no validation** — see pitfalls.

**Validator classes** remain for hooks, schemas built from injected services, or `validatorCache`. They are middleware with `{ validator: true }`, returning a Zod schema from `json()`:

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { ValidationService } from '@asenajs/ergenecore';
import { z } from 'zod';

@Middleware({ validator: true })
export class CreateUserValidator extends ValidationService {
  json() {
    return z.object({ name: z.string().min(1), email: z.string().email() });
  }
}
```

Attach via the route option: `@Post({ path: '/', validator: CreateUserValidator })`. A route uses a contract **or** a `validator`, never both (boot fails). Validator schema factories (`json()`, …) run **once per route at startup** by default; set `validatorCache: false` on the route when a factory must rebuild its schema per request. A validation failure throws a 400 `HttpException` subclass — `ContractValidationError` for a contract, the adapter's `ValidationError` for a validator class — that reaches your `@Catch(HttpException)` (below); match it with `isValidationError()`.

## Server config

One `@Config` class per app, `extends ConfigService` (adapter import). Methods: `serveOptions()`, `globalMiddlewares()`, `transport()`. Its `onError()` / `onNotFound()` are **deprecated (removed in 1.0)** — never write new code against them; error handling lives in an `@ExceptionHandler` class.

## Error handling

An `@ExceptionHandler` class is an ordinary IoC component (`@Inject` works). Its `@Catch(...types)` methods answer thrown errors; a `@NotFound()` method answers unmatched routes:

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

- **Most specific wins**: the thrown value's prototype chain is walked upward (the `instanceof` test) — `@Catch(NotFoundException)` beats `@Catch(HttpException)` beats `@Catch(Error)`. One `@Catch` per exception type, one `@Catch()` catch-all and one `@NotFound()` per application; a duplicate fails the boot.
- `@Catch()` (catch-all) is not `@Catch(Error)`: only the catch-all receives non-`Error` throws (`throw 'boom'`) — type its parameter `unknown`.
- **Chain**: `@Catch` → deprecated `@Config.onError` → adapter default (`@NotFound()` → `@Config.onNotFound` → adapter 404). Returning nothing **declines** to the next link; on HTTP a non-`Response` value also declines (logged as an error); a handler that throws is logged and declines — the original error continues (`@MessagePattern` differs, below).
- **Scopes** (`ctx.scope`): `'http'`; `'microservice'` — for `@MessagePattern` (`kind: 'message'`) a returned value becomes a **successful** reply (the caller's `send()` resolves with it, even `{ ok: false }` or `null`), a returned `Response` is declined with a warning, returning `undefined` is how the caller gets to see the failure (`REMOTE_ERROR`), and an error the `@Catch` throws replaces the original one (a sanitized error for the caller); `'event'` (`@On`) and `@EventPattern` (`kind: 'event'`) are **observe-only** — the original failure continues (retry/DLQ, default error log) and a returned value is ignored with a warning. Always narrow on `ctx.scope` before touching `ctx.context`.
- A failure set off by work a running `@Catch` started (an event it emitted) is not routed back to `@Catch` — in-process only, not across a real broker.
- Throw `HttpException` or a ready-made status exception from `@asenajs/asena/adapter` — works on both adapters: `throw new NotFoundException({ error: 'User not found' })`, `throw new HttpException(429, 'Too Many Requests')`.
- **Always** test exceptions with `isHttpException()` inside a handler; **never** `instanceof HttpException` (duplicate module copies silently break instanceof — see pitfalls). `@Catch(HttpException)` uses the same brand.
- On the Hono adapter, if you need Hono's `HTTPException`, import it from `@asenajs/hono-adapter` — **never** from `hono/http-exception`.
- Don't re-wrap `error.message` that is already a JSON string — double-encoding trap.

## Testing quickstart

Everything imports from `@asenajs/asena/test`; use Bun's test runner (`bun:test`) only.

| Level | Tool | Boots |
|---|---|---|
| Unit (one class) | `mockComponent` | Nothing — auto-mocks all injected deps |
| Web layer | `createWebTest` | Real controllers/middleware/validators, rest auto-mocked |
| Full app | `createTestApp` | Whole IoC container + real HTTP server |

```typescript
import { mockComponent } from '@asenajs/asena/test';

const { instance, mocks } = mockComponent(AuthService);
mocks.userService.createUser.mockResolvedValue({ id: 'u1' });
```

```typescript
import { createTestApp, silentLogger } from '@asenajs/asena/test';
import { createErgenecoreAdapter } from '@asenajs/ergenecore';
import { mock } from 'bun:test';

await using app = await createTestApp({
  adapter: createErgenecoreAdapter({ logger: silentLogger }),
  components: [UserController, UserService],   // roots only — the @Inject(Class) closure comes along
  overrides: { UserService: { getAll: mock(async () => [{ id: '1' }]) } },
});

await app.get('/users').expectStatus(200).expectJson([{ id: '1' }]);
```

- `overrides` keys are **registered component names**. Never `Object.assign(instance, { dep: mock })` — it throws.
- `components` needs only the **roots**: every class reachable through `@Inject(SomeClass)` is registered automatically. A dependency injected **by name** (`@Inject('UserService')`) has no class to follow — list it or override it, or the boot fails with `createTestApp: missing dependencies:` naming the component and field.
- Default `port: 0` (kernel-assigned free port) — never hardcode test ports.

## CLI quickstart

| Command | Effect |
|---|---|
| `asena create [name] --adapter=<hono\|ergenecore>` | Scaffold a project |
| `asena generate controller\|service\|middleware\|validator\|config\|exception-handler\|websocket` (`asena g c\|s\|m\|v\|config\|eh\|ws`) | Generate a component |
| `asena console` | Run the app in development with a browser console: component graph, runtime, endpoint tester, requests, messaging, config, exceptions. Needs core 0.13+; refuses `NODE_ENV=production` |
| `asena build` | Production bundle → `dist/index.asena.js`. Does NOT rewrite your entry file; a hand-written `components:` array is yours and wins |
| `asena init` | Create `asena-config.ts` |
| `asena doctor` | Read-only project check (decorator flags, duplicate copies, peer ranges, minify settings, auth wiring); exit 1 on failure |

Output paths and class-name suffixes come from `asena-config.ts` — see `references/cli.md`.

## Critical pitfalls

- **Duplicate module copies** — the #1 silent killer. Two copies of `@asenajs/asena` or `hono` in `node_modules` break every `instanceof`-based match: deliberate 401/403 `HttpException`s collapse into 500s, DI and `@OnStart` misbehave with no error. Cause: a dependency (or stale lockfile) pinning its own copy; `hono`/`zod` are peers precisely so the app owns exactly one copy. Diagnose: `bun pm ls --all | grep hono` (or `grep @asenajs/asena`) — more than one entry is the bug, except a copy nested under `@asenajs/asena-cli` 0.11.0 (see the version matrix), which the app never loads. Fix: `rm -rf node_modules bun.lock && bun install`.
- **tsconfig flags missing** — without `experimentalDecorators` + `emitDecoratorMetadata` components are silently not registered; the server boots with zero routes.
- **Entry-file ordering** — a component declared below `AsenaServerFactory.create()` is not registered (warning logged).
- **`globalMiddlewares` as a property** — silently ignored; must be a method.
- **Throwing in `@OnStart`** aborts boot by design — don't "fix" it by swallowing errors; fix the cause or move non-critical work out.
- **`ulak.send`/`ulak.emit` inside `@OnStart`** throws `NO_TRANSPORT` — the transport isn't connected yet; defer the first publish until after startup.
- **`ws.publish()` excludes the sender** — send to the sender explicitly with `ws.send()`. Rooms are not enumerable — keep your own registry if you need a member list.
- **Validation errors bypass your controller** — they reach `@Catch(HttpException)` (or a `@Catch(ContractValidationError, ValidationError)`); match with `isValidationError()`, don't expect them in the route body.
- **A catch-all that always answers 500** — `@Catch()` and `@Catch(Error)` also receive `ValidationError` and `HttpException` when no narrower `@Catch` claims them, turning every 400/401/404 into a 500 (an info line at boot names the handler). Pair it with `@Catch(HttpException)`, or return nothing for `isHttpException(error)`.
- **A `@Catch` answering in an event scope** — returning a value for an `@On` or `@EventPattern` failure does nothing (a warning is logged); the failure still retries / dead-letters / logs. Recover inside the event handler itself if the event must not fail.
- **A routing 404 is not an error** — an unmatched route never reaches `@Catch`, not even the catch-all; answer it with `@NotFound()`. A missing *record* is a throw (`NotFoundException`).
- **A `@Catch` answer is logged at debug only** — once a handler returns a `Response`, the adapter skips its own error log and the framework writes the original error at `debug`. Log inside the handler (unexpected errors especially).
- **Contract class erased to a type** — `import type { CreateUserContext }` (or `emitDecoratorMetadata` off) leaves no runtime metadata: the route registers **without validation** and `context.input` throws. Import the contract class as a value; also never read `context.input` in middleware, the constructor or a field initializer — it exists only once validation finished.
- **Auth markers are inert on their own** — `@Protected` / `@Public` / `@Roles` from `@asenajs/asena/auth` only record metadata; core never enforces them. Without an auth implementation that reads them, every route stays public. Don't treat the decorator as protection. Fix: install `@asenajs/asena-auth`, register a provider (`@Auth` or `@BetterAuth`) and pass `imports: [...authComponents]` to `AsenaServerFactory.create()` — its `AuthGuard` enforces the marks (`references/auth.md`). `asena doctor`'s `auth-enforcement` check fails when the marks are imported but no guard is. On a `@WebSocket` class the marks are ignored even then — use `AuthWebSocketMiddleware`.
- **Rate limiter behind a proxy** — since adapter 5.0.0 `RateLimiterMiddleware` keys on the connecting peer and ignores `X-Forwarded-For` unless `trustProxy` is set. Behind a reverse proxy, without `trustProxy` every client shares the proxy's one bucket.
- **`CorsMiddleware` is not a WebSocket origin policy** — a disallowed origin still upgrades. Put `WebSocketOriginMiddleware` before any cookie-based auth in `@WebSocket({ middlewares })`.
- **Validator factory reading request-time state** — with the default `validatorCache: true`, `json()`/`query()` run once at startup; state read inside the factory is frozen. Read mutable state in a `refine`/hook, or set `validatorCache: false`.
- **Duplicate routes fail the boot** — `/x` and `/x/` for the same method (or two controllers resolving to the same canonical path) now throw at startup instead of one silently shadowing the other.
- **Cross-adapter divergences were closed in adapter 4.0.0** — `getQuery` returns `undefined` for an absent key and `setResponseHeader` replaces on BOTH adapters, and both have `appendResponseHeader`. On an adapter `3.x` project the old split still applies (ergenecore `''` / hono `undefined`; hono's `setResponseHeader` appends), so check the installed version before relying on either. Prefer `?? default` over `|| default` for `getQuery`.
- **The Redis cache decorators are silently inert** until a `@RedisCache` class extending `CachePostProcessor` exists in `src/` — methods still run, Redis is just never touched. `@Transaction` used to fail the same way; since asena-drizzle 3.1 it **fails the boot** instead (`@Transaction methods are not wrapped: ...`), which also fires when a transactional class sits inside a post-processor's dependency closure.
- **A class in a PostProcessor's dependency closure is never post-processed** — Phase A constructs it before post-processing is active. Any decorator implemented by a processor silently stops applying to it. Keep processor dependencies minimal.

## Feature → reference routing

Read the reference before writing code in its area:

| Working on | Read |
|---|---|
| Route params, cookies, streaming, static files, HTML pages | `references/controllers-and-context.md` |
| `@Value`, `imports`, scopes, `@Strategy`, lifecycle, shutdown/signals, `server.resolve()`, inheritance, `@PostProcessor` | `references/dependency-injection-and-lifecycle.md` |
| CORS, rate limiting, auth middleware, route context contracts, Zod validators, `validatorCache` | `references/middleware-and-validation.md` |
| Authentication: `@Auth`, `@BetterAuth`, `@Protected`/`@Roles` enforcement, sessions, Bearer/WebSocket tokens, `TestAuthProvider` | `references/auth.md` |
| `@Config`, `@ExceptionHandler` / `@Catch` / `@NotFound`, `HttpException` and status exceptions, validation failures, shutdown, deployment | `references/configuration-and-errors.md` |
| WebSocket namespaces, rooms, multi-pod, Ulak broker | `references/websocket-and-ulak.md` |
| In-process events (`@On`), cron (`@Schedule`) | `references/events-and-scheduling.md` |
| `@MessagePattern`, Redis Streams, Kafka (client + transport), headless services | `references/microservices.md` |
| Drizzle repositories, transactions, Redis caching | `references/database-and-caching.md` |
| Logging, OpenTelemetry tracing | `references/observability.md` |
| OpenAPI generation (validators or contracts), Swagger UI / Scalar docs page | `references/openapi.md` |
| Unit/web/full-app tests | `references/testing.md` |
| Scaffolding, generate, build config, suffixes, `asena doctor`, `asena console` | `references/cli.md` |
| Adapter choice, built-in middleware, Hono migration | `references/adapters.md` |

## Full documentation lookup

For any API not covered above, fetch the official docs as raw markdown: `https://asena.sh/raw/<path>.md` — the complete page index with descriptions is in `references/docs-map.md` (machine index: `https://asena.sh/llms.txt`). Prefer the bundled references first; fetch raw docs for deeper detail.
