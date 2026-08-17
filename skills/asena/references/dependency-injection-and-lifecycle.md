# AsenaJS Dependency Injection & Lifecycle

Field-based IoC wiring (`@Inject`/`@Strategy`), component scopes, `@OnStart`/`@OnStop` ordering and failure policy, inheritance rules, and `@PostProcessor` — as of `@asenajs/asena` 0.10.x.

## Contents

- @Inject Forms
- Injection Rules
- @Strategy and @Implements
- Component Scopes
- Lifecycle: @OnStart / @OnStop
- Stopping the Server and Signal Handling
- keepAlive and Headless Workers
- server.resolve()
- Inheritance
- @PostProcessor

## @Inject Forms

```typescript
import { Inject } from '@asenajs/asena/decorators/ioc';
import { Controller, Service } from '@asenajs/asena/decorators';

@Controller('/users')
export class UserController {
  @Inject(UserService)              // class token - type-safe, preferred
  private userService: UserService;

  @Inject('UserService')            // string token - the registered component name
  private byName: UserService;

  @Inject(DatabaseService, (s: DatabaseService) => s.connection)   // expression form
  private db: BunSQLDatabase;
}
```

Expression signature: `@Inject(ServiceClass, (service) => any)` — `ServiceClass` is a class or a string; the optional expression transforms the injected instance before assignment. It can extract a property (`(s) => s.connection`) or a call result (`(s: ItemService) => s.getItems()`). Keep expressions shallow; deep chains like `(s) => s.getConfig().db.primary.connection.pool` are a maintenance trap.

A string token resolves the component's **registered name**: the class name by default, or the name given to `@Service('UserService')`. **Warning:** the Bun bundler may rename classes during a production build, changing the default container key — pin an explicit `@Service('Name')` whenever string injection is used.

## Injection Rules

- **Field injection only.** Injected fields are `undefined` inside the constructor. Do setup in `@OnStart`, never in `constructor()`.
- **Injected fields are read-only accessors.** `@Inject` and `@Strategy` install accessors with no setter — assignment throws, naming the field and class. In tests use the test app's `overrides` option or `mockComponent()`, never `Object.assign(instance, { dep: fake })`.
- **`@Inject` resolves components, not values.** There is no value/constant registry: `@Inject('ENV_API_KEY')` throws `ENV_API_KEY is not registered` at startup. Read plain configuration from `process.env` (or a `@Service` that wraps it).
- **No circular dependencies.** `A → B → A` breaks the graph; extract shared logic into a third service both inject. For the WebSocket-service cycle specifically, inject a `ulak('/namespace')` handle instead of the `@WebSocket` class (see `https://asena.sh/raw/concepts/ulak.md`).
- **Inside a component, always use `@Inject`.** Resolving by hand works but hides the dependency from the graph, so the container can no longer order construction around it.

## @Strategy and @Implements

`@Implements('Key')` registers a component under an interface key (producer side); `@Strategy('Key')` injects **every** implementation registered under that key as an array (consumer side).

```typescript
import { Implements, Strategy } from '@asenajs/asena/decorators/ioc';
import { Service } from '@asenajs/asena/decorators';

export interface NotificationService {
  send(userId: string, message: string): Promise<void>;
}

@Service()
@Implements('NotificationService')
export class EmailNotificationService implements NotificationService {
  async send(userId: string, message: string) { /* ... */ }
}

@Service()
export class NotificationManager {
  @Strategy('NotificationService')
  private channels: NotificationService[];   // ALWAYS an array

  async notifyUser(userId: string, message: string) {
    for (const channel of this.channels) await channel.send(userId, message);
  }
}
```

| `@Implements('Key')` components | Injected value |
|:--------------------------------|:---------------|
| 0 | `[]` — a normal state, **not an error** |
| 1 | `[implementation]` |
| 2+ | every implementation |

- Zero implementations is legitimate (a plugin point with no plugins yet), so a **mistyped key cannot fail at boot** — it just stays `[]` forever. The container logs one `debug`-level line per empty injection site at startup (`[IocEngine] Strategy key '...' has no implementations - ... will be injected as []`). Enable `debug` on the logger when a strategy collection is unexpectedly empty. Keys supplied through a test's `overrides` are not reported.
- Expression form exists here too: `@Strategy('PaymentProvider', (p: PaymentProvider) => p.name)` injects `string[]`.
- **Never** `@Inject('SomeInterfaceKey')`: an interface key with multiple implementations hands back the array, and the failure surfaces later as `... is not a function` on first use. Interface keys are for `@Strategy`.
- **Migration (≤0.9.0):** an empty key used to abort boot with `<key> is not registered`, and exactly one implementation injected a bare instance instead of an array — drop any workarounds for either.

## Component Scopes

```typescript
import { Service } from '@asenajs/asena/decorators';
import { Scope } from '@asenajs/asena/decorators/ioc';

@Service()                                              // Scope.SINGLETON (default)
export class BasicService {}

@Service('MyCustomService')                             // named (string-injection key)
export class NamedService {}

@Service({ scope: Scope.PROTOTYPE })                    // new instance per injection
export class PrototypeService {}

@Service({ name: 'RequestHandler', scope: Scope.PROTOTYPE })
export class RequestHandlerService {}
```

`ComponentParams`: `{ name?: string; scope?: Scope }` — `name` defaults to the class name, `scope` to `Scope.SINGLETON`.

| | `SINGLETON` (default) | `PROTOTYPE` |
|:--|:--|:--|
| Instances | One per application | One per injection |
| State | Shared across app | Isolated per injection |
| Use for | Stateless services, configs, connections, caches | Per-request/mutable state, isolation |

- Singleton instances are shared across all requests — never keep per-request mutable state on one (`this.currentUser = ...` is a race condition). Pass state as parameters, or use `PROTOTYPE`.
- Only singletons take part in shutdown. A `PROTOTYPE` component's `@OnStart` still runs at construction, but declaring `@OnStop` on one produces a boot warning — the container keeps no handle on transient instances, so the hook can never fire.

## Lifecycle: @OnStart / @OnStop

```typescript
import { Service } from '@asenajs/asena/decorators';
import { Inject, OnStart, OnStop } from '@asenajs/asena/decorators/ioc';

@Service()
export class PriceFeed {
  @Inject(MarketClient)
  private market: MarketClient;

  private subscription?: Subscription;

  @OnStart()
  public async start() {
    this.subscription = await this.market.subscribe('prices');  // deps are injected here
  }

  @OnStop()
  public async stop() {
    await this.subscription?.close();
  }
}
```

When `@OnStart` runs (from `server.start()`): every component is constructed, every `@Inject`/`@Strategy` field is populated, every `@PostProcessor` has run — but application setup has **not** happened: `@Config` hooks are unread, microservice transports unconnected, routes unregistered, HTTP socket unbound, scheduled jobs not started. Consequences:

- `ulak.send()` / `ulak.emit()` inside `@OnStart` throws `NO_TRANSPORT` — defer the first publish. `@OnStop` has no such limit (transports are torn down after stop hooks).
- No request can outrun a start hook: the socket binds only after every hook returned.
- A `@Config` that builds a transport from an injected service finds that service already started.

### Start sequence

1. Components constructed; `@Inject`/`@Strategy` wired; `@PostProcessor`s run
2. **`@OnStart` hooks run**, in registration order (topologically sorted — dependencies before dependents)
3. `@Config` hooks are read and applied
4. Microservice transports connect and start listening
5. Event handlers and scheduled jobs are registered
6. Routes, frontend controllers and WebSocket namespaces registered with the adapter
7. Adapter binds the HTTP socket
8. Scheduled jobs start
9. Health endpoint starts, signal handlers installed

### Stop sequence (`server.stop()`)

1. Signal handlers and keep-alive released; lifecycle state → `STOPPING` (`/ready` answers `503` for the whole drain)
2. Scheduled jobs stop
3. Adapter stops (closes HTTP socket and WebSocket transport)
4. **`@OnStop` hooks run**, in reverse start order
5. Microservice transports drain and disconnect
6. `ulak.dispose()` (automatic — do not call it yourself)
7. Health endpoint stops last

Every stop step is contained: a throwing step is logged, the rest still run, and collected failures are raised together as an `AggregateError`.

### Ordering rules

- Start walks components in registration order; stop walks the same list backwards — a component still has its dependencies while releasing its own resources.
- Within one class, multiple `@OnStart` methods run in declaration order (ancestors first when inherited); `@OnStop` methods run in the reverse. An inherited hook runs **once**, however deep the chain.

### Failure policy

| | Throws | Hangs |
|:--|:--|:--|
| `@OnStart` | **Boot aborts.** Started components are rolled back, state → `FAILED`, `server.start()` rejects with an error naming the hook and carrying the original as `cause` | No timeout — a hook that never resolves is a `start()` that never resolves. A run-loop component starts its loop and returns (`void this.loop()`), it does not await it |
| `@OnStop` | Logged and skipped; shutdown continues | Bounded per hook (default `5000` ms), then logged and skipped |

Use `@OnStart` for setup that must succeed at boot (open a pool, subscribe to a topic); validate recoverable input elsewhere. A failed boot does **not** stop the server for you — transports already connected stay up:

```typescript
try {
  await server.start();
} catch (error) {
  await server.stop();   // safe on a server that never finished starting
  throw error;
}
```

- Only components whose start hook completed are stopped; rolled-back or never-started components are skipped.
- `stop()` runs once: concurrent calls share one teardown, later calls are no-ops returning the original outcome.
- A `@Config` may carry `@OnStart`: it runs before its config hooks are read, so it can prepare what `transport()` or `globalMiddlewares()` uses.

### `@PostConstruct` is a deprecated alias

`@PostConstruct` still works — it writes the same metadata as `@OnStart`; renaming the import is the whole migration. But the **timing changed in 0.10.0** (breaking):

- It used to run inside `Container.register()`, mid-scan. It now runs from `server.start()` — a component resolved from a server that was **created but never started is not initialised**.
- A throwing hook no longer calls `process.exit(1)`; it throws and `server.start()` rejects.
- Start hooks now run **after** `@PostProcessor`s, against the post-processed instance.
- Exception: `@PostProcessor`s and their dependencies keep construction-time `@OnStart` timing (see below).

## Stopping the Server and Signal Handling

```typescript
await server.stop();                          // closeActiveConnections: true
await server.stop(false);                     // wait for in-flight connections
await server.stop({ drainTimeout: 30_000 });  // give transports 30s to drain
```

| `stop()` option | Type | Default | Description |
|:--|:--|:--|:--|
| `closeActiveConnections` | `boolean` | `true` | Close in-flight connections instead of waiting |
| `drainTimeout` | `number` | transport's own | Drain window for microservice transports, ms |
| `hookTimeout` | `number` | `shutdown.timeout`, else `5000` | Per-`@OnStop` ceiling, ms |

Types are exported: `import { LifecycleState } from '@asenajs/asena'; import type { AsenaStopOptions, ShutdownOptions } from '@asenajs/asena';`

Signal handling is **on by default**: `start()` installs handlers, `stop()` removes them (no listener accumulation across test boots).

```typescript
const server = await AsenaServerFactory.create({
  adapter, logger, port: 3000,
  shutdown: {
    signals: true,            // default: SIGTERM, SIGINT, SIGHUP
    timeout: 5000,            // per-@OnStop ceiling, ms
    forceExitAfter: false,    // number (ms) forces process exit after signal-triggered shutdown
    onUnhandledError: false,  // treat uncaught exception/rejection as shutdown request
  },
});
```

| `shutdown` option | Type | Default | Notes |
|:--|:--|:--|:--|
| `signals` | `boolean \| NodeJS.Signals[]` | `true` | List narrows; `false` = own the signals yourself (needed if the entry file installs its own handlers, or a signal triggers two shutdowns) |
| `timeout` | `number` | `5000` | Per-`@OnStop` ceiling; `stop({ hookTimeout })` overrides per call |
| `forceExitAfter` | `number \| false` | `false` | Deadline on the **process**, not on `stop()`; timer is `unref()`'d; forced exit is non-zero |
| `onUnhandledError` | `boolean` | `false` | Log, stop, exit non-zero — swallows the top-level stack, hence opt-in |

A second signal during a running shutdown exits immediately with code `130`.

## keepAlive and Headless Workers

A process with no listening socket exits once `start()` resolves. `keepAlive` holds the event loop open; defaults to `true` in headless mode, `false` otherwise (an HTTP server is held by its own socket). This lets the run loop live in a component instead of the entry file:

```typescript
@Service()
export class OutboxWorker {
  private running = false;
  private loop?: Promise<void>;

  @OnStart()
  public start() {
    this.running = true;
    this.loop = this.run();   // returns immediately; the loop keeps running
  }

  @OnStop()
  public async stop() {
    this.running = false;
    await this.loop;          // let the current batch finish
  }

  private async run() {
    while (this.running) { /* drain a batch */ await Bun.sleep(250); }
  }
}
```

```typescript
// src/index.ts
const server = await AsenaServerFactory.create({
  headless: true,           // no HTTP adapter
  logger,
  health: { port: 9090 },   // /healthz/live + /healthz/ready probes
  // keepAlive defaults to true here
});
await server.start();
```

`SIGTERM` reaches `stop()`, `stop()` reaches `@OnStop`, the worker finishes its batch. Health probe details: `https://asena.sh/raw/concepts/lifecycle.md`.

## server.resolve()

For code that is **not** a component (entry file, migration runner, one-off script) — `@Inject` only works inside components:

```typescript
const server = await AsenaServerFactory.create({ adapter, logger });
await server.start();

const feed = await server.resolve<PriceFeed>('PriceFeed');
await feed.warmUp();
```

- The name is the registered name (class name by default, or the `@Service('name')` string) — same key `@Inject('...')` takes.
- **Resolve after `start()`, not after `create()`** — in between, the component is constructed and injected but its `@OnStart` has not run (pool unopened, cache empty).
- **An unknown name throws** `<name> is not registered`; there is no `undefined` to check for.
- **A name shared by two classes resolves to an array** (the container promotes duplicate names).
- `server.coreContainer.container.resolve()` still works but `server.resolve()` is the supported spelling.

## Inheritance

The rule: **what changes the behaviour of a request travels with the route; what says where the route lives, or what a class is, belongs to the concrete class.**

**An undecorated subclass is not a component.** `@Controller`, `@Service`, `@Middleware`, `@Config`, `@MessageController` mark the class that carries them — a subclass of a `@Controller` is not a controller unless decorated too, and since 0.9.0 it is not registered at all. If a class "disappears" after upgrading, it was missing its decorator.

| Inherited ✅ | Not inherited ❌ (class identity) |
|:--|:--|
| `@Get`/`@Post`/`@Put`/`@Delete`/`@Patch`/`@All` (mount under the **subclass's** `@Controller` prefix) | `@Controller`/`@WebSocket`/`@FrontendController` path |
| `@Page`, `@On`, `@MessagePattern`/`@EventPattern` (subclass's prefix/transport) | Component name and scope — two classes must not resolve to one name |
| `@Inject`, `@Strategy` (ancestor fields are injected) | `@EventService`/`@MessageController` prefix and transport |
| `@Implements` — subclass joins its base's interface key, so `@Strategy` lists pick it up without redeclaring | Component type (`CONTROLLER`, `SERVICE`, …) |
| `@OnStart`/`@OnStop` — run once per method; start hooks ancestors-first, stop reversed | `@Hidden` on a class (`@asenajs/asena-openapi`) |
| `@Override`, `@Hidden` on a method (`@asenajs/asena-openapi`), `@Transaction` | |
| Controller-level `middlewares` — unioned across the chain, ancestors first; a subclass may **add**, never drop | |
| Plain methods, getters, statics | |

- **Overrides resolve by method name**: a subclass method with the same name replaces the inherited route entry (repeating the decorator is optional — only needed to change path/options). Same rule for `@On`, `@MessagePattern`, `@EventPattern`, `@Page`.
- **Conflicts fail the build by resolved path**: two *differently named* methods on the same resolved route throw `Duplicate route detected: GET /probe/live — already registered by ...` and the server refuses to start. Same for two handlers on one `@MessagePattern` (request/response is ambiguous); `@EventPattern` and `@On` allow fan-out.
- To publish base-class routes *without* the base's guards, move the routes to an undecorated base and let each concrete controller declare its own `middlewares`.
- Inherited handlers are logged at startup (`Controller ProbeController inherits routes: ...`). **Warning:** an inherited `@EventPattern`/`@MessagePattern` opens a real subscription on the configured transport — extending a base class means consuming its patterns.
- Asena never scans `node_modules`: shared components ship as undecorated base classes in a package; the decorated concrete subclass lives in the app's `src/`.

## @PostProcessor

Cross-cutting instance interception during IoC bootstrap (Spring `BeanPostProcessor` analogue). For single-component init, use `@OnStart` instead.

```typescript
import { PostProcessor } from '@asenajs/asena/decorators';
import type { ComponentPostProcessor } from '@asenajs/asena/ioc/types';

@PostProcessor()                       // accepts { name?: string }
export class TracingPostProcessor implements ComponentPostProcessor {
  postProcess<T>(instance: T, Class: any): T {
    // instance: all @Inject/@Strategy fields populated; @OnStart has NOT run yet
    // return a wrapped/modified instance, or the original unchanged
    // (null/undefined keeps the original); T | Promise<T> allowed
    return instance;
  }
}
```

Timing per component: `constructor → @Inject → @Strategy → postProcess() → (later, from server.start()) @OnStart`. Multiple processors chain in FIFO registration order.

Bootstrap runs in two phases: **Phase A** creates PostProcessors and their `@Inject` dependencies — none of which are post-processed themselves (keep processor dependencies minimal); **Phase B** creates everything else through `postProcess()`. Phase A components run their `@OnStart` at **construction** (old timing): it cannot reach microservice transports, and their `@OnStop` runs on `server.stop()` even if `start()` was never called.
