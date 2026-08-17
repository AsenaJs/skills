---
name: asena
description: Expert guidance for building AsenaJS applications - the IoC web framework for the Bun runtime with Spring Boot-style decorators and field-based dependency injection. Use when creating or modifying AsenaJS projects or any code importing @asenajs packages - HTTP controllers and routing, services, middleware, Zod validation, WebSocket namespaces, microservices over Redis Streams or Kafka, Redis caching, in-process events, scheduled cron tasks, static files, Drizzle ORM repositories, OpenAPI generation, OpenTelemetry tracing, testing with mockComponent/createTestApp/createWebTest, or the asena CLI. Covers @asenajs/asena, @asenajs/ergenecore, @asenajs/hono-adapter and all official packages. Triggers - asena, asenajs, AsenaServerFactory, createErgenecoreAdapter, createHonoAdapter, ergenecore, hono-adapter, @Controller, @Service, @Inject, @OnStart, @WebSocket, @MessagePattern, @StaticServe, @FrontendController, AsenaLogger, ulak, asena-config.
version: 0.1.0
license: MIT
---

# AsenaJS

AsenaJS is a decorator-based IoC web framework that runs ONLY on Bun. It looks like NestJS or Spring Boot but is neither: decorator names, import paths, adapter factories, and lifecycle hooks all differ. Never write AsenaJS code from memory of other frameworks — follow this file for the core rules and read the matching file in `references/` (routing table at the bottom) before writing code in any feature area.

## Version matrix

Package versions are NOT aligned — do not assume one version number across packages. As of this skill's sync (2026-08):

| Package | npm | Version | Peer deps |
|---|---|---|---|
| Core framework | `@asenajs/asena` | 0.10.1 | — (`reflect-metadata` only runtime dep) |
| CLI | `@asenajs/asena-cli` | 0.10.0 | — |
| Hono adapter | `@asenajs/hono-adapter` | 3.1.0 | `hono`, `zod` |
| Ergenecore adapter (native Bun) | `@asenajs/ergenecore` | 3.1.0 | `zod` |
| Redis (cache + transports) | `@asenajs/asena-redis` | 3.1.0 | `redis` |
| Kafka (client + transport) | `@asenajs/asena-kafka` | 3.0.0 | `kafkajs` |
| OpenAPI | `@asenajs/asena-openapi` | 2.0.0 | `zod` |
| OpenTelemetry | `@asenajs/asena-otel` | 2.0.0 | `@opentelemetry/*` SDK family |
| Drizzle ORM | `@asenajs/asena-drizzle` | 3.0.0 | `drizzle-orm` (+ `pg` or `mysql2` per driver) |
| Logger | `@asenajs/asena-logger` | 2.0.0 | — |

Every add-on package also peers `@asenajs/asena` (`^0.10.0`) and `reflect-metadata`. Requires **Bun >= 1.3.12**. There is no Node.js support — both adapters bind to `Bun.serve()`.

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

All **core** decorators come from `@asenajs/asena` subpaths. Only the `Context` type, the middleware/validator/config base classes, and adapter built-ins come from the adapter package (`@asenajs/ergenecore` or `@asenajs/hono-adapter`). Add-on packages export their own decorators: `@Database`/`@Repository`/`@Drizzle`/`@Transaction` (`@asenajs/asena-drizzle`), `@Redis`/`@RedisCache` (`@asenajs/asena-redis`), `@Kafka` (`@asenajs/asena-kafka`), `@Otel` (`@asenajs/asena-otel`), `@Hidden` (`@asenajs/asena-openapi`).

| Import from | Symbols |
|---|---|
| `@asenajs/asena` | `AsenaServerFactory`, `AsenaServer`, `LifecycleState` |
| `@asenajs/asena/decorators` | `Controller`, `Service`, `Middleware`, `Config`, `WebSocket`, `Schedule`, `EventService`, `MessageController`, `FrontendController`, `PostProcessor`, `StaticServe`, `Override`, `Component` (also re-exported here) |
| `@asenajs/asena/decorators/http` | `Get`, `Post`, `Put`, `Patch`, `Delete`, `Head`, `Options`, `All`, `Route`, `Page` |
| `@asenajs/asena/decorators/ioc` | `Inject`, `Strategy`, `Implements`, `OnStart`, `OnStop`, `Component`, `Scope` |
| `@asenajs/asena/adapter` | `HttpException`, `isHttpException`, `isValidationError`, adapter SPI types |
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
| adapter package | `Context` (type), `MiddlewareService`, `ValidationService`, `ConfigService`, built-in middlewares (`CorsMiddleware`, …) |

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
- `@Inject` resolves registered components only — there is no value/token registry for arbitrary strings. Read env config from `process.env`. Exception: framework factory tokens (`ulak('/chat')`, `ulak.messages('svc')`, `emitter()`) are valid `@Inject` arguments — they resolve framework-provided components.
- `@Strategy` always yields an **array**: zero implementations gives `[]`, not an error. Guard for empty.
- **Always** `await` every promise in handlers and lifecycle methods — an unawaited rejected promise shuts the whole server down.
- If you bundle/minify with identifier mangling, register by explicit string name (`@Service('UserService')`, `@Inject('UserService')`) — class-name-derived registration breaks under mangling.
- Use `server.resolve()` only outside components and only after `server.start()`; inside components always use `@Inject`.

## Architecture discipline

Asena promotes a strict layered architecture: `Request → Controller → Service → Repository → Database`. Enforce these boundaries in every file you generate or modify — violations here are the most common quality failure in generated code:

- **Controllers stay thin** — parse the request, call a service, shape the response. **Never** put business logic, direct repository/database access, or multi-service orchestration in a controller.
- **A repository belongs to exactly one domain service.** Inject `OrderRepository` only into `OrderService`. **Never** inject a repository into another domain's service or into a controller — that bypasses the owning domain's rules and couples the caller to the schema.
- **Cross-domain access goes through the owning service.** When `InvoiceService` needs order data, inject `OrderService`, not `OrderRepository`. If two services would end up needing each other, extract the shared logic into a third service, or decouple the reaction with an in-process event (`@On`).
- **Services own the business invariants.** Stock checks, state transitions, ownership rules live in the service — not in the controller and not in the repository. Validators (`ValidationService`) check request *shape*; services check *meaning*.
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
    // reject: throw HttpException. pass: await next()
    await next();
  }
}
```

Four attachment levels: global (returned from `globalMiddlewares()` in the `@Config` class), pattern-based include/exclude, controller-level (`@Controller({ path, middlewares: [...] })`), route-level (`@Get({ path, middlewares: [...] })`).

- `globalMiddlewares()` **must be a method** on the `@Config` class. As a property it is silently ignored.
- Adapter built-ins (`CorsMiddleware`, `RateLimiterMiddleware`, …) must be subclassed with `@Middleware()` before use — never list them directly: `@Middleware() export class GlobalCors extends CorsMiddleware {}`.

Validators are middleware with `{ validator: true }`, returning a Zod schema from `json()`:

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

Attach via the route option: `@Post({ path: '/', validator: CreateUserValidator })`. Once an `onError` handler exists, validation failures reach it as `ValidationError` — match with `isValidationError()`.

## Server config and errors

One `@Config` class per app, `extends ConfigService` (adapter import). Available methods: `serveOptions()`, `onError(error, context)`, `onNotFound()`, `globalMiddlewares()`, `transport()`.

```typescript
import { Config } from '@asenajs/asena/decorators';
import { isHttpException } from '@asenajs/asena/adapter';
import { ConfigService, type Context } from '@asenajs/ergenecore';

@Config()
export class ServerConfig extends ConfigService {
  public onError(error: Error, context: Context) {
    if (isHttpException(error)) return error.getResponse();
    return context.send({ message: 'Internal server error' }, 500);
  }
}
```

- Throw `HttpException` from `@asenajs/asena/adapter` — it works on both adapters: `throw new HttpException(404, { message: 'User not found' })`.
- **Always** match exceptions with `isHttpException()`; **never** `instanceof HttpException` (duplicate module copies silently break instanceof — see pitfalls).
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
  components: [UserController, UserService],   // explicit list — no filesystem scan
  overrides: { UserService: { getAll: mock(async () => [{ id: '1' }]) } },
});

await app.get('/users').expectStatus(200).expectJson([{ id: '1' }]);
```

- `overrides` keys are **registered component names**. Never `Object.assign(instance, { dep: mock })` — it throws.
- Default `port: 0` (kernel-assigned free port) — never hardcode test ports.

## CLI quickstart

| Command | Effect |
|---|---|
| `asena create [name] --adapter=<hono\|ergenecore>` | Scaffold a project |
| `asena generate controller\|service\|middleware\|validator\|config\|websocket` (`asena g c\|s\|m\|v\|config\|ws`) | Generate a component |
| `asena build` | Production bundle → `dist/index.asena.js` |
| `asena init` | Create `asena-config.ts` |

Output paths and class-name suffixes come from `asena-config.ts` — see `references/cli.md`.

## Critical pitfalls

- **Duplicate module copies** — the #1 silent killer. Two copies of `@asenajs/asena` or `hono` in `node_modules` break every `instanceof`-based match: deliberate 401/403 `HttpException`s collapse into 500s, DI and `@OnStart` misbehave with no error. Cause: a dependency (or stale lockfile) pinning its own copy; `hono`/`zod` are peers precisely so the app owns exactly one copy. Diagnose: `bun pm ls --all | grep hono` (or `grep @asenajs/asena`) — more than one entry is the bug. Fix: `rm -rf node_modules bun.lock && bun install`.
- **tsconfig flags missing** — without `experimentalDecorators` + `emitDecoratorMetadata` components are silently not registered; the server boots with zero routes.
- **Entry-file ordering** — a component declared below `AsenaServerFactory.create()` is not registered (warning logged).
- **`globalMiddlewares` as a property** — silently ignored; must be a method.
- **Throwing in `@OnStart`** aborts boot by design — don't "fix" it by swallowing errors; fix the cause or move non-critical work out.
- **`ulak.send`/`ulak.emit` inside `@OnStart`** throws `NO_TRANSPORT` — the transport isn't connected yet; defer the first publish until after startup.
- **`ws.publish()` excludes the sender** — send to the sender explicitly with `ws.send()`. Rooms are not enumerable — keep your own registry if you need a member list.
- **Validation errors bypass your controller** — they surface in `onError` as `ValidationError` once a handler exists; match with `isValidationError()`, don't expect them in the route body.
- **Cross-adapter divergences are silent** — a missing query key resolves to `''` on ergenecore but `undefined` on hono-adapter (treat falsy as absent); `setResponseHeader()` replaces on ergenecore but appends on hono-adapter. Never encode one adapter's behavior when the project might run the other.
- **`@Transaction` and the Redis cache decorators are silently inert** until their PostProcessor subclass exists in `src/`: `@Drizzle` class extending `TransactionPostProcessor` (asena-drizzle), `@RedisCache` class extending `CachePostProcessor` (asena-redis). Methods still run — the database/Redis is just never touched.

## Feature → reference routing

Read the reference before writing code in its area:

| Working on | Read |
|---|---|
| Route params, cookies, streaming, static files, HTML pages | `references/controllers-and-context.md` |
| Scopes, `@Strategy`, lifecycle, shutdown/signals, `server.resolve()`, inheritance, `@PostProcessor` | `references/dependency-injection-and-lifecycle.md` |
| CORS, rate limiting, auth middleware, Zod validators | `references/middleware-and-validation.md` |
| `@Config`, `HttpException`, shutdown, deployment | `references/configuration-and-errors.md` |
| WebSocket namespaces, rooms, multi-pod, Ulak broker | `references/websocket-and-ulak.md` |
| In-process events (`@On`), cron (`@Schedule`) | `references/events-and-scheduling.md` |
| `@MessagePattern`, Redis Streams, Kafka (client + transport), headless services | `references/microservices.md` |
| Drizzle repositories, transactions, Redis caching | `references/database-and-caching.md` |
| Logging, OpenTelemetry tracing | `references/observability.md` |
| Swagger / OpenAPI generation | `references/openapi.md` |
| Unit/web/full-app tests | `references/testing.md` |
| Scaffolding, generate, build config, suffixes | `references/cli.md` |
| Adapter choice, built-in middleware, Hono migration | `references/adapters.md` |

## Full documentation lookup

For any API not covered above, fetch the official docs as raw markdown: `https://asena.sh/raw/<path>.md` — the complete page index with descriptions is in `references/docs-map.md` (machine index: `https://asena.sh/llms.txt`). Prefer the bundled references first; fetch raw docs for deeper detail.
