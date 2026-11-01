---
name: asena
description: Expert guidance for building AsenaJS applications - the Bun IoC web framework with Spring Boot-style decorators and field-based dependency injection. Use when creating or modifying AsenaJS projects or any code importing @asenajs packages - controllers and routing, services, middleware, validation (route context contracts, Zod validators), authentication and sessions, WebSocket namespaces, microservices over Redis Streams or Kafka, Redis caching, events, cron tasks, static files, Drizzle repositories, OpenAPI, OpenTelemetry, testing with mockComponent/createTestApp/createWebTest, or the asena CLI. Triggers - asena, asenajs, Context.define, AsenaServerFactory, createErgenecoreAdapter, createHonoAdapter, ergenecore, hono-adapter, asena-auth, @Controller, @Service, @Inject, @Value, @OnStart, @Protected, @Auth, @BetterAuth, AuthGuard, authComponents, requireAuthSession, @WebSocket, @MessagePattern, @StaticServe, @FrontendController, AsenaLogger, ulak, asena-config, asena doctor.
version: 0.4.0
license: MIT
---

# AsenaJS

AsenaJS is a decorator-based IoC web framework that runs ONLY on Bun. It looks like NestJS or Spring Boot but is neither: decorator names, import paths, adapter factories, and lifecycle hooks all differ. Never write AsenaJS code from memory of other frameworks — follow this file for the core rules and read the matching file in `references/` (routing table at the bottom) before writing code in any feature area.

## Version matrix

Package versions are NOT aligned — do not assume one version number across packages. As of this skill's sync (2026-11-01, the asena-auth 1.0.0 release). If an installed package resolves lower than the table shows, the behaviour this file describes for it may not apply — check before relying on it.

| Package | npm | Version | Peer deps |
|---|---|---|---|
| Core framework | `@asenajs/asena` | 0.12.3 | — (`reflect-metadata` only runtime dep) |
| CLI | `@asenajs/asena-cli` | 0.12.0 | — (no `@asenajs/asena` dependency; reads the project's own core) |
| Hono adapter | `@asenajs/hono-adapter` | 5.0.3 | `hono`, `zod`, core `^0.12.0` |
| Ergenecore adapter (native Bun) | `@asenajs/ergenecore` | 5.0.2 | `zod`, core `^0.12.0` |
| Auth (sessions, Better Auth, route guards) | `@asenajs/asena-auth` | 1.0.0 | core `^0.12.2`; `better-auth` optional (Better Auth provider only) |
| Redis (cache + transports) | `@asenajs/asena-redis` | 4.0.1 | `redis` |
| Kafka (client + transport) | `@asenajs/asena-kafka` | 4.0.1 | `kafkajs` |
| OpenAPI | `@asenajs/asena-openapi` | 3.2.0 | `zod`, core `^0.12.0` |
| OpenTelemetry | `@asenajs/asena-otel` | 3.0.1 | `@opentelemetry/*` SDK family |
| Drizzle ORM | `@asenajs/asena-drizzle` | 4.0.1 | `drizzle-orm` (+ `pg` or `mysql2` per driver) |
| Logger | `@asenajs/asena-logger` | 3.0.1 | — |

Both adapters and `asena-openapi` peer `@asenajs/asena` `^0.12.0`; `asena-auth` peers `^0.12.2` (its docs require core 0.12.3+ with ergenecore 5.0.2+ or hono-adapter 5.0.3+), is adapter-agnostic, and ships a `@asenajs/asena-auth/testing` subpath; the other add-ons (redis, kafka, otel, drizzle, logger) peer `^0.11.0 || ^0.12.0`. `reflect-metadata` comes in through core. `asena-cli` 0.11.0 depends on `@asenajs/asena` `^0.11.0` (0.11.1 and later have no core dependency): on a 0.12 project it nests a second copy at `node_modules/@asenajs/asena-cli/node_modules/@asenajs/asena` that app code never resolves but `asena doctor`'s `duplicate-packages` check reports — upgrade the CLI to remove it. Every package requires **Bun >= 1.4.0**. There is no Node.js support — both adapters bind to `Bun.serve()`.

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
| `@asenajs/asena/decorators` | `Controller`, `Service`, `Middleware`, `Config`, `WebSocket`, `Schedule`, `EventService`, `MessageController`, `FrontendController`, `PostProcessor`, `StaticServe`, `Override`, `Component` (also re-exported here) |
| `@asenajs/asena/decorators/http` | `Get`, `Post`, `Put`, `Patch`, `Delete`, `Head`, `Options`, `All`, `Route`, `Page` |
| `@asenajs/asena/decorators/ioc` | `Inject`, `Value`, `Strategy`, `Implements`, `OnStart`, `OnStop`, `Component`, `Scope` |
| `@asenajs/asena/adapter` | `HttpException`, `isHttpException`, `isValidationError`, adapter SPI types |
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
| adapter package | `Context` (type and value — `Context.define()`), `MiddlewareService`, `ValidationService`, `ConfigService`, built-in middlewares (`CorsMiddleware`, `RateLimiterMiddleware`, `WebSocketOriginMiddleware`, …) |

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
    // reject: throw HttpException. pass: await next()
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

Attach via the route option: `@Post({ path: '/', validator: CreateUserValidator })`. A route uses a contract **or** a `validator`, never both (boot fails). Validator schema factories (`json()`, …) run **once per route at startup** by default; set `validatorCache: false` on the route when a factory must rebuild its schema per request. Once an `onError` handler exists, validation failures from either style reach it as `ValidationError` — match with `isValidationError()`.

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
| `asena generate controller\|service\|middleware\|validator\|config\|websocket` (`asena g c\|s\|m\|v\|config\|ws`) | Generate a component |
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
- **Validation errors bypass your controller** — they surface in `onError` as `ValidationError` once a handler exists; match with `isValidationError()`, don't expect them in the route body.
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
| `@Config`, `HttpException`, shutdown, deployment | `references/configuration-and-errors.md` |
| WebSocket namespaces, rooms, multi-pod, Ulak broker | `references/websocket-and-ulak.md` |
| In-process events (`@On`), cron (`@Schedule`) | `references/events-and-scheduling.md` |
| `@MessagePattern`, Redis Streams, Kafka (client + transport), headless services | `references/microservices.md` |
| Drizzle repositories, transactions, Redis caching | `references/database-and-caching.md` |
| Logging, OpenTelemetry tracing | `references/observability.md` |
| OpenAPI generation (validators or contracts), Swagger UI / Scalar docs page | `references/openapi.md` |
| Unit/web/full-app tests | `references/testing.md` |
| Scaffolding, generate, build config, suffixes, `asena doctor` | `references/cli.md` |
| Adapter choice, built-in middleware, Hono migration | `references/adapters.md` |

## Full documentation lookup

For any API not covered above, fetch the official docs as raw markdown: `https://asena.sh/raw/<path>.md` — the complete page index with descriptions is in `references/docs-map.md` (machine index: `https://asena.sh/llms.txt`). Prefer the bundled references first; fetch raw docs for deeper detail.
