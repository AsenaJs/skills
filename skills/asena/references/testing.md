# Testing

Testing utilities for AsenaJS (as of `@asenajs/asena` 0.10.x): unit-level dependency mocking plus two integration harnesses, all exported from `@asenajs/asena/test`, built exclusively for Bun's test runner.

## Contents

- Choosing a level
- Ground rules
- mockComponent / mockComponentAsync (unit)
- createWebTest (controller slice)
- createTestApp (full application)
- Fluent HTTP assertions
- Dispatch modes
- createTestApp known behaviours
- Full CRUD example
- WebSocket unit test

## Choosing a level

| Utility | Use when | Container | Adapter |
|---|---|---|---|
| `mockComponent` | Testing one class's logic in isolation | Bypassed | None |
| `createWebTest` | Testing a controller's routing, middlewares, validation | Real; non-web deps auto-mocked | Real |
| `createTestApp` | Testing the whole application end to end | Real | Real |

`mockComponent` is fastest and covers most service-level tests. Reach for a harness only when the thing under test *is* framework behaviour: a route matching, a middleware short-circuiting, a validator rejecting a payload.

```typescript
// All utilities come from the same subpath
import {
  mockComponent, mockComponentAsync, createMockFromClass, createDeepMock,
  createTestUlakStub, createTestApp, createWebTest, silentLogger,
} from '@asenajs/asena/test';
```

**Warning:** `@asenajs/asena/test` imports `bun:test` at module scope — it runs only under `bun test`, never Jest/Vitest.

`createMockFromClass`/`createDeepMock` are the lower-level factories `mockComponent` builds on — prefer `mockComponent`; reach for them only to mock a bare class outside the DI graph.

## Ground rules

- Always clean up with `await using app = await createTestApp(...)` — `TestApp` implements `Symbol.asyncDispose`; `app.stop()` is idempotent, no `afterEach` needed.
- Always silence boot logs: pass `silentLogger` to the adapter factory (harness `logger` already defaults to `silentLogger`).
- Always leave `port` at its default `0` — the kernel hands out a guaranteed-free port; read it back from `app.port` / `app.baseUrl`. Never hand-roll random ports.
- Always key harness `overrides` by the REGISTERED service name: `@Service('Mailer')` → `overrides: { Mailer: double }` — never by the injected field name.
- NEVER assign to an injected field. `@Inject` and `@Strategy` install accessors with no setter, so `Object.assign(instance, { dep: fake })` throws (the error names the field and class). Use `overrides`, or `mockComponent` for a unit-level double.
- Never override controllers: a plain object carries no `@Controller` metadata, so its routes would never register. Override the services the controller injects instead.
- Never override core services (`Container`, `ServerLogger`, `__Ulak__`, `EventEmitter`, …) — they are wired during bootstrap phases 1–5 and have already captured their dependencies; attempting it throws.
- Displacing one member of a `@Strategy` array by overriding the interface name is not supported.

## mockComponent / mockComponentAsync (unit)

Instantiates a component directly — no container, no server — with every `@Inject`ed dependency auto-mocked from IoC metadata.

```typescript
function mockComponent<T extends object>(
  ComponentClass: new (...args: any[]) => T,
  options?: MockComponentOptions
): MockedComponent<T>

async function mockComponentAsync<T extends object>(...): Promise<MockedComponent<T>>

interface MockComponentOptions {
  injections?: string[];              // only mock these fields; others stay undefined
  overrides?: Record<string, any>;    // custom doubles, keyed by FIELD name
  postConstruct?: (instance: any) => void | Promise<void>; // YOUR callback, not @OnStart
}

interface MockedComponent<T> {
  instance: T;                 // component with mocks injected
  mocks: Record<string, any>;  // keyed by FIELD name
}
```

```typescript
import { describe, test, expect } from 'bun:test';
import { mockComponent } from '@asenajs/asena/test';

describe('AuthService', () => {
  test('registers a user', async () => {
    const { instance, mocks } = mockComponent(AuthService);

    mocks.userService.createUser.mockResolvedValue({ id: 'user-123', name: 'John' });

    const result = await instance.register('John', 'john@example.com', 'pass');

    expect(result.user.id).toBe('user-123');
    expect(mocks.userService.createUser).toHaveBeenCalledWith('John', 'john@example.com');
  });
});
```

### What each field receives

| Injection | Auto-mock |
|---|---|
| `@Inject(UserService)` | Object shaped like the class — every method a `bun:test` mock; async methods resolve `null`, sync return `undefined` |
| `@Inject(ulak('/chat'))` and other expression injections | Expression evaluated against a deep mock — every property access yields a Bun mock, every call chains, all assertable |
| `@Inject('UserService')` | Plain `{}` — a string carries no class, so no shape can be derived. Pass an override yourself |

```typescript
// String injections need explicit doubles
mockComponent(LegacyService, {
  overrides: { userService: { findById: mock(async () => ({ id: '1' })) } },
});
```

### Option semantics

- `injections: ['stripe']` — only `mocks.stripe` is defined; unlisted fields stay `undefined`.
- `overrides` is the **final** injected value. For expression injections (`@Inject(UserService, (s) => s.createUser)`, `ulak(...)`) the expression is skipped entirely and your override is used as-is. Presence is checked with `Object.hasOwn`, so falsy overrides (`0`, `''`, `null`, `undefined`) are injected, not ignored.
- `postConstruct` is **your** callback, run after injection. `mockComponent` never runs the component's own `@OnStart`/`@OnStop` (nothing goes through container or server). Call the hook explicitly if you want it, and use `mockComponentAsync` when the callback is async:

```typescript
const { instance } = await mockComponentAsync(DatabaseService, {
  postConstruct: async (inst) => inst.onStart(),  // the @OnStart method, invoked by you
});
```

- Inheritance works: dependencies declared on a base class are mocked alongside the subclass's own.
- Expression injections are real Bun mocks after evaluation: `mocks.createUserFn.mockResolvedValue({ id: 'user-123' })`.

### Ulak (WebSocket messaging) doubles

`@Inject(ulak('/path'))` fields work with zero setup — the namespace becomes a deep mock. For a fully typed stub, use `createTestUlakStub`, which implements the whole `Ulak.NameSpace` interface (`broadcast`, `to`, `toSocket`, `toMany`, `getSocketCount`) with Bun mocks:

```typescript
const statsChannel = createTestUlakStub('/ws/public/stats');
const { instance } = mockComponent(UserService, { overrides: { statsChannel } });

await instance.createAnonUser('John');
expect(statsChannel.broadcast).toHaveBeenCalledWith({ action: 'update', data: { newUser: 1 } });
```

## createWebTest (controller slice)

Boots **only the web layer** (Spring's `@WebMvcTest`): the controllers you pass, their middlewares and validators run for real; every other injected dependency becomes a generated mock shaped like the real class.

```typescript
interface WebTestOptions {
  adapter: AsenaAdapter;              // required
  controllers: Class | Class[];       // required - only @Controller classes accepted here
  components?: Class[];               // extra REAL components (services, middlewares, ...)
  overrides?: Record<string, object>; // explicit doubles; win over auto-mocks
  logger?: ServerLogger;
  port?: number;
  dispatch?: 'server' | 'socket';
}
// returns { app, mocks } - app is a full TestApp (same fluent client, stop(), await using)
```

```typescript
import { createWebTest, silentLogger } from '@asenajs/asena/test';
import { createErgenecoreAdapter } from '@asenajs/ergenecore';

test('returns a user', async () => {
  const adapter = createErgenecoreAdapter({ logger: silentLogger });

  const { app, mocks } = await createWebTest({ adapter, controllers: [UserController] });

  mocks.UserService.findById.mockResolvedValue({ id: '1', name: 'Ada' });

  await app.get('/api/users/1').expectStatus(200).expectJson({ id: '1', name: 'Ada' });
  expect(mocks.UserService.findById).toHaveBeenCalledWith('1');

  await app.stop();
});
```

What runs for real: the controllers you pass, controller-level and route-level `middlewares`, `validator`, `staticServe` (all resolved by name at start-up — missing ones throw), anything in `components`, and core services. Everything else is auto-mocked.

### The `mocks` object

- Keyed by **service name**, not field name: `mocks.UserService` ✅, `mocks.userService` ❌ (`mockComponent` keys by field name — this is the one place they differ).
- Exactly **one** mock per service, shared by every component that injects it — configure it once, assert anywhere.
- Explicit `overrides` also appear in `mocks`, so it is the single place to reach every double.
- `@Inject('Name')` string injections cannot be shaped: the mock falls back to `{}` and a warning is logged — pass an explicit override for those services.

### Promotion and overrides

```typescript
// Promote a mock back to the real class: list it in components - it disappears from mocks
const { mocks } = await createWebTest({
  adapter,
  controllers: [UserController],
  components: [UserService],   // now real
});
mocks.UserService; // undefined

// Explicit overrides always beat the generated mock
const double = { findById: mock(async () => ({ id: '1' })) };
const { mocks: m } = await createWebTest({
  adapter, controllers: [UserController], overrides: { UserService: double },
});
expect(m.UserService).toBe(double);
```

Known behaviours:

- **Validators are real.** A route validator requiring a UUID makes `app.get('/api/users/1')` return **400**, not 200 — production behaviour, and the point of a slice test.
- **`@OnStart` runs against mocks.** A real component whose dependency was auto-mocked sees async mock methods resolve `null` during its start hook.
- **Only `@Controller` classes go in `controllers`** — services and middlewares go through `components`.

## createTestApp (full application)

Boots a **complete** application (Spring's `@SpringBootTest`): IoC container, every bootstrap phase, the adapter's real routing pipeline.

```typescript
interface TestAppOptions {
  adapter: AsenaAdapter;              // required
  components: Class[];                // required - skips filesystem scanning entirely
  overrides?: Record<string, object>; // service name -> replacement instance
  logger?: ServerLogger;              // default: silentLogger
  port?: number;                      // default: 0 (Bun picks a free port)
  dispatch?: 'server' | 'socket';     // default: 'server'
}
```

- `components` must name every class the app needs at start-up: controllers, services, middlewares, validators, configs. Nothing is scanned from disk.
- `app.port` / `app.baseUrl` expose the bound address.

### Overrides (Spring's `@MockBean`)

`overrides` maps a registered service name to a replacement instance. Overrides are seeded **before** any user component registers, so:

- the real class is never constructed — its `@OnStart` (and deprecated `@PostConstruct` alias) never runs;
- every dependent captures the double, because injection closures are built eagerly at registration time.

```typescript
const userService = { getAll: mock(async () => [{ id: '1', name: 'Ada' }]) };

await using app = await createTestApp({
  adapter,
  components: [UserController, UserService],
  overrides: { UserService: userService },
});

await app.get('/api/users').expectStatus(200).expectJson([{ id: '1', name: 'Ada' }]);
expect(userService.getAll).toHaveBeenCalledTimes(1);
```

Overridable: services, repositories, plain components. Not overridable: core services (throws), controllers (routes lost — override their services instead), individual `@Strategy` array members. A component registered as `@Service('Mailer')` is overridden by **that** name.

### Container access

```typescript
const service = await app.resolve<UserService>('UserService');
expect(app.container.has('UserRepository')).toBe(true);
```

## Fluent HTTP assertions

`app.get()` / `.post()` / … return a `TestHttpCall`. Nothing is sent until awaited; assertions run in chained order; the send is memoized (awaiting twice does not re-issue the request).

| Method | Assertion |
|---|---|
| `expectStatus(code)` | Status code; failure message includes method, URL and response body |
| `expectHeader(name, value)` | Header equals a string or matches a `RegExp` |
| `expectJson(expected)` | Whole JSON body deep-equals `expected` |
| `expectJsonContains(partial)` | JSON body contains at least these properties |
| `expectBody(expected)` | Raw text body equals a string or matches a `RegExp` |
| `expect(fn)` | Escape hatch — receives the buffered response |

Awaiting resolves to a `TestHttpResponse` with a pre-buffered body — readable repeatedly, any way:

```typescript
const response = await app.get('/api/users').expectStatus(200);

expect(response.json<User[]>()).toHaveLength(3);
expect(response.text()).toContain('Ada');
expect(response.headers.get('x-total')).toBe('3');
expect(response.raw).toBeInstanceOf(Response);
```

## Dispatch modes

- `'server'` (default): listens on a TCP port — identical to production.
- `'socket'`: listens on a **unix domain socket** — still the adapter's real pipeline (real router, cookies, validators, WebSocket upgrades) but no TCP port, so parallel suites can never collide. `app.port` is `0`; `app.socketPath` matches `/\.sock$/`.

```typescript
await using app = await createTestApp({ adapter, components, dispatch: 'socket' });
await app.get('/api/users').expectStatus(200);

// WebSocket URLs differ per mode - always build them with app.wsUrl()
const socket = new WebSocket(app.wsUrl('/ws/chat'));
// 'server' -> ws://localhost:43117/ws/chat
// 'socket' -> ws+unix:///tmp/asena-test-1234-1.sock:/ws/chat
```

**Warning:** socket dispatch requires an adapter honouring `AsenaStartOptions.unix` — `@asenajs/hono-adapter` and `@asenajs/ergenecore` both do.

## createTestApp known behaviours

- **Cron runs for real.** `cronRunner.startAll()` executes at start-up — a `@Schedule` component in the test set will fire.
- **`@OnStart` / `@OnStop` run for real.** Boot calls `server.start()`, cleanup calls `server.stop()` — that is what releases pools, subscribers and timers between test files.
- **A throwing `@OnStart` fails the boot, not the process.** `createTestApp()` rejects with an error naming the hook (up to 0.9.x the container called `process.exit(1)` instead, reporting `0 pass / 1 fail` with no cause).
- **No signal listeners, no held loop.** The harness passes `shutdown: { signals: false }` and `keepAlive: false` — booting twenty apps in one suite installs nothing global.

## Full CRUD example

```typescript
import { createTestApp, silentLogger } from '@asenajs/asena/test';
import { createErgenecoreAdapter } from '@asenajs/ergenecore';
import { describe, test, expect, mock } from 'bun:test';

describe('User API', () => {
  test('creates a user', async () => {
    const adapter = createErgenecoreAdapter({ logger: silentLogger });

    await using app = await createTestApp({
      adapter,
      components: [AppConfig, UserController, UserService, CreateUserValidator],
    });

    await app
      .post('/api/users', { body: JSON.stringify({ name: 'Ada', email: 'ada@example.com' }) })
      .expectStatus(201)
      .expectJsonContains({ name: 'Ada' });
  });

  test('surfaces repository failures as 500', async () => {
    const adapter = createErgenecoreAdapter({ logger: silentLogger });

    await using app = await createTestApp({
      adapter,
      components: [AppConfig, UserController, UserService],
      overrides: {
        UserService: { getAll: mock(async () => { throw new Error('database is down'); }) },
      },
    });

    await app.get('/api/users').expectStatus(500);
  });
});
```

**Note:** with the Hono adapter, the factory returns a tuple — `const [adapter] = createHonoAdapter({ logger: silentLogger })`.

## WebSocket unit test

`@WebSocket` services extend `AsenaWebSocketService`; test their handlers with `mockComponent` and a hand-stubbed `Socket`:

```typescript
import { describe, test, expect, mock } from 'bun:test';
import { mockComponent } from '@asenajs/asena/test';
import type { Socket } from '@asenajs/asena/web-socket';

describe('ChatSocket', () => {
  test('onMessage saves and broadcasts', async () => {
    const { instance, mocks } = mockComponent(ChatSocket);

    const mockSocket = { data: { values: { userId: 'user-123' } } } as unknown as Socket;
    instance.to = mock(() => {});  // inherited from AsenaWebSocketService - not an injection

    await instance.onMessage(mockSocket, 'Hello!');

    expect(mocks.messageService.saveMessage).toHaveBeenCalledWith('user-123', 'Hello!');
    expect(instance.to).toHaveBeenCalledWith('user:user-123', { type: 'message', data: 'Hello!' });
  });
});
```

For handler-context stubs generally: mock only the `Context` methods the handler calls (`getParam`, `getBody`, `send`, `setValue`, …) and cast with `as unknown as Context`. Deeper detail: `https://asena.sh/raw/testing/examples.md`.
