# Auth

Authentication with `@asenajs/asena-auth` (1.0.x): native server-side sessions (`@Auth`) and Better Auth (`@BetterAuth`) behind core's `@asenajs/asena/auth` contract. The contract's `@Protected` / `@Public` / `@Roles` marks become a route guard that answers 401 / 403 before route middlewares and validation run — identical on both adapters.

## Contents

- Install and imports
- Native quick start
- Guards and roles
- Reading the session
- Session lifecycle
- Optional sessions on public routes
- Bearer tokens and WebSocket
- Better Auth
- Multiple providers
- Options
- Session stores
- Testing
- Pitfalls

## Install and imports

```bash
bun add @asenajs/asena-auth               # native sessions
bun add @asenajs/asena-auth better-auth   # plus the Better Auth provider
```

Requires Bun >= 1.4.0. Peers `@asenajs/asena` `^0.12.2` and, optionally, `better-auth` `>=1.2.0 <2`; the docs require ergenecore 5.0.2+ or hono-adapter 5.0.3+. Zero runtime dependencies — tokens and password hashing use Bun built-ins, and the package never imports `better-auth`, so an app without that provider does not install it.

| Import from | Symbols |
|---|---|
| `@asenajs/asena/auth` (core contract) | `Protected`, `Public`, `Roles`, `getAuthSession`, `requireAuthSession`, `AuthUser`, `AuthSession`, `AsenaAuthProvider`, `AUTH_PROVIDER_KEY` |
| `@asenajs/asena-auth` | `Auth`, `AsenaAuthService`, `BetterAuth`, `BetterAuthService`, `BetterAuthController`, `authComponents`, `AuthGuard`, `AuthGuardMiddleware`, `AuthProviderRegistry`, `AuthSessionMiddleware`, `AuthWebSocketMiddleware`, `applySessionCookie`, `clearSessionCookie`, `hashPassword`, `verifyPassword`, `MemoryAuthStore`, `AuthSessionStore` (type) |
| `@asenajs/asena-auth/testing` | `TestAuth`, `TestAuthProvider` |

## Native quick start

The snippets import `Context` and `ConfigService` from `@asenajs/ergenecore`; on the Hono adapter import both from `@asenajs/hono-adapter`. Every duration option is in **seconds**.

```typescript
// src/auth/AppAuth.ts
import { AsenaAuthService, Auth } from '@asenajs/asena-auth';

@Auth({ sessionTtl: 3600 })
export class AppAuth extends AsenaAuthService {}
```

The provider registers under its class name (`AppAuth`) — that is its `@Inject`/`overrides`/`app.resolve()` name. Sessions live in an in-process `MemoryAuthStore` until you pass a `store`.

Issue a session at login. `createSession(user)` returns `{ token, session }`; `applySessionCookie` writes the token as the `asena_session` cookie for browsers, and the token in the body serves `Authorization: Bearer` clients:

```typescript
// src/controllers/AuthController.ts
import { Controller } from '@asenajs/asena/decorators';
import { Post } from '@asenajs/asena/decorators/http';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { applySessionCookie } from '@asenajs/asena-auth';
import type { Context } from '@asenajs/ergenecore'; // hono: '@asenajs/hono-adapter'
import { AppAuth } from '../auth/AppAuth';
import { UserService } from '../services/UserService';

@Controller('/auth')
export class AuthController {
  @Inject(AppAuth)
  private auth!: AppAuth;

  @Inject(UserService)
  private userService!: UserService;

  @Post('/login')
  async login(context: Context) {
    const { email, password } = await context.getBody<{ email: string; password: string }>();
    const user = await this.userService.verifyCredentials(email, password);

    if (!user) return context.send({ error: 'Invalid email or password' }, 401);

    const { token } = await this.auth.createSession({ id: user.id, roles: user.roles });
    await applySessionCookie(context, this.auth, token);

    return context.send({ token });
  }
}
```

- The user passed to `createSession` needs an `id` and is stored as a plain-data snapshot — keep it small; a function inside it throws a `TypeError`.
- `verifyCredentials` is your service. Hash and check with the provider: `await this.auth.hashPassword(password)` / `await this.auth.verifyPassword(password, hash)` (bcrypt cost 12 by default; `hashing: { algorithm: 'argon2id' }` switches, and verification reads the algorithm from the hash, so old hashes keep verifying). Standalone `hashPassword` / `verifyPassword` exports do the same.

Register the package components. The guard and middlewares ship inside the package, and the scan never walks `node_modules`, so `imports` is the only way in:

```typescript
// src/index.ts
import { AsenaServerFactory } from '@asenajs/asena';
import { authComponents } from '@asenajs/asena-auth';

const server = await AsenaServerFactory.create({ adapter, logger, imports: [...authComponents] });

await server.start();
```

| Component | Role |
|---|---|
| `AuthGuard` | Post-processor: reads the marks at boot and puts the guard in front of every protected route |
| `AuthGuardMiddleware` | The guard itself: 401 / 403 |
| `AuthProviderRegistry` | Finds the providers, picks the primary, validates every protected route at boot |
| `AuthSessionMiddleware` | Optional global middleware — sessions on public routes |
| `AuthWebSocketMiddleware` | Authenticates WebSocket upgrades |

Without `AuthGuard` in the container nothing enforces the marks. Registry imported without the guard logs `[asena-auth] AuthGuard is not imported: @Protected/@Roles routes are not enforced and not validated.`

Protect a route:

```typescript
// src/controllers/MeController.ts
import { Protected, requireAuthSession } from '@asenajs/asena/auth';
import { Controller } from '@asenajs/asena/decorators';
import { Get } from '@asenajs/asena/decorators/http';
import type { Context } from '@asenajs/ergenecore';

@Protected()
@Controller('/me')
export class MeController {
  @Get('/')
  async me(context: Context) {
    const { user } = requireAuthSession(context);

    return context.send({ id: user.id, roles: user.roles });
  }
}
```

Anonymous → **401** with `WWW-Authenticate: Bearer`. A request carrying both a Bearer token and the session cookie is resolved by the Bearer token.

## Guards and roles

```typescript
import { Protected, Public, Roles } from '@asenajs/asena/auth';
import { Controller } from '@asenajs/asena/decorators';
import { Delete, Get } from '@asenajs/asena/decorators/http';
import type { Context } from '@asenajs/ergenecore';

@Protected()
@Controller('/projects')
export class ProjectController {
  @Get('/')
  list(context: Context) {
    return context.send({ projects: [] });
  }

  @Roles('admin')
  @Delete('/:id')
  remove(context: Context) {
    return context.send({ deleted: context.getParam('id') });
  }

  @Public()
  @Get('/featured')
  featured(context: Context) {
    return context.send({ projects: [] });
  }
}
```

- A class mark covers every route; a method mark **replaces** it for that route. Roles of a class mark never flow down to a method with its own mark; the provider name does (`@Roles('admin')` under `@Protected({ provider: 'Partner' })` still asks `Partner`).
- `@Roles(...)` implies `@Protected` and is any-of. A session holding none of the roles gets **403** (no `WWW-Authenticate`). `@Roles()` with no role throws at decoration time.
- Marks stacked on one target: the decorator written on top wins.
- Marks are inherited; a `@Public()` subclass opens what its `@Protected` base guards.
- Marks apply only to `@Controller` classes. On a service or a `@WebSocket` namespace they are ignored with a boot warning — sockets use `AuthWebSocketMiddleware`.

Execution order per protected route, both adapters:

```text
global middlewares → controller middlewares → auth guard → route middlewares → validator / Context.define contract → handler
```

An anonymous request never reaches route middlewares, validation or the handler: an invalid body from an anonymous client is a 401, not a 400. Global and controller-level middlewares do run before the guard.

**Roles resolver.** The default reads `user.roles` (array), else comma-splits `user.role` (the Better Auth admin plugin's shape), else `[]`. The `roles` option replaces it — here, a role hierarchy and a typed user:

```typescript
import type { AuthUser } from '@asenajs/asena/auth';
import { AsenaAuthService, Auth } from '@asenajs/asena-auth';

export interface AppUser extends AuthUser {
  email: string;
  roles: string[];
}

const implied: Record<string, string[]> = {
  owner: ['owner', 'admin', 'member'],
  admin: ['admin', 'member'],
};

@Auth({
  roles: ({ user }) => (user.roles ?? []).flatMap((role) => implied[role] ?? [role]),
})
export class AppAuth extends AsenaAuthService<AppUser> {}
```

The type parameter types `createSession(user)`; handlers pass it to the readers: `requireAuthSession<AppUser>(context).user.email`.

## Reading the session

Core's `getAuthSession(context)` reads without deciding; `requireAuthSession(context)` returns the session or throws a 401 `HttpException` (on `undefined` and `null` alike).

| `getAuthSession` returns | Meaning |
|---|---|
| an `AuthSession` | A provider resolved a session for this request |
| `null` | Resolved as anonymous — e.g. by `AuthSessionMiddleware` on a public route |
| `undefined` | Nothing resolved it: the route is not protected and no session middleware ran |

`AuthSession` is `{ user, session: { id, userId, createdAt, expiresAt } }`. `user` is what `createSession` stored (or what `loadUser` returned). For a native session `session.id` is the token's SHA-256 — safe to show the user, and what `revokeSessionById` takes.

## Session lifecycle

```typescript
// src/controllers/AccountController.ts
import { Protected, requireAuthSession } from '@asenajs/asena/auth';
import { Controller } from '@asenajs/asena/decorators';
import { Delete, Post } from '@asenajs/asena/decorators/http';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { clearSessionCookie } from '@asenajs/asena-auth';
import type { Context } from '@asenajs/ergenecore';
import { AppAuth } from '../auth/AppAuth';

@Protected()
@Controller('/account')
export class AccountController {
  @Inject(AppAuth)
  private auth!: AppAuth;

  @Post('/logout')
  async logout(context: Context) {
    await this.auth.logout(context);

    return context.send({ ok: true });
  }

  @Post('/rotate')
  async rotate(context: Context) {
    return context.send({ token: await this.auth.rotateSession(context) });
  }

  @Delete('/sessions/:id')
  async revoke(context: Context) {
    const { user } = requireAuthSession(context);
    const id = context.getParam('id');
    const own = await this.auth.listUserSessions(user.id);

    if (!own.some(({ session }) => session.id === id)) {
      return context.send({ error: 'Not found' }, 404);
    }

    await this.auth.revokeSessionById(id);

    return context.send({ ok: true });
  }

  @Post('/logout-everywhere')
  async logoutEverywhere(context: Context) {
    await this.auth.revokeUserSessions(requireAuthSession(context).user.id);
    await clearSessionCookie(context, this.auth);

    return context.send({ ok: true });
  }
}
```

| Method | Effect |
|---|---|
| `createSession(user)` | Stores a new session; returns `{ token, session }` |
| `logout(ctx)` | Ends the request's session (Bearer token, else cookie) and clears the cookie; without a session it only clears the cookie |
| `rotateSession(ctx, user?)` | Replaces the request's session, returns the new token, re-issues the cookie when the old token came from it. Throws without a session |
| `listUserSessions(userId)` | The user's live sessions with the stored user; needs a store with `listByUser` |
| `revokeSessionById(id)` | Ends one session by `session.id` — one device |
| `revokeSession(token)` | Ends one session by its token |
| `revokeUserSessions(userId)` | Ends every session of the user |
| `getSession(request)` | Resolves a raw `Request` — what the guard calls |
| `cookieName` / `cookieOptions` | The session cookie as configured |

- A revoked session is a 401 on its very next request — every request reads the store.
- **Rotate after a privilege change.** Rotation is a fresh authentication: `createdAt`, and with it `absoluteTtl`, restart. Without `user` the new session keeps the stored user. `getAuthSession(ctx)` returns the new session for the rest of the request.
- **An id from the client is untrusted.** `revokeSessionById` revokes whatever id it gets — check ownership first, as above.

## Optional sessions on public routes

Guards resolve the session on protected routes only. To read it on public routes, register `AuthSessionMiddleware` globally:

```typescript
import { Config } from '@asenajs/asena/decorators';
import { AuthSessionMiddleware } from '@asenajs/asena-auth';
import { ConfigService } from '@asenajs/ergenecore';

@Config()
export class AppConfig extends ConfigService {
  public globalMiddlewares() {
    return [AuthSessionMiddleware];
  }
}
```

A public handler then reads `getAuthSession(context)?.user.id ?? null`. The middleware never rejects (anonymous → `null`; guards still decide), asks the provider once per request (a following guard reuses the result), uses the primary or only provider, and re-issues a slid session cookie on public routes too.

## Bearer tokens and WebSocket

A token is 32 random bytes (`tokenBytes`) in unpadded base64url — 43 characters, valid as a header, cookie and subprotocol value. `Authorization: Bearer <token>` (scheme case-insensitive) wins over the cookie; a Bearer session slides in the store, but no cookie is written for it. `bearer: false` makes the provider cookie-only **and** turns off the WebSocket subprotocol token, which arrives as that same header.

WebSocket: put `AuthWebSocketMiddleware` on the namespace. The session lands in `ws.data.values.authSession`:

```typescript
import type { AuthSession } from '@asenajs/asena/auth';
import { WebSocket } from '@asenajs/asena/decorators';
import { AsenaWebSocketService, type Socket } from '@asenajs/asena/web-socket';
import { AuthWebSocketMiddleware } from '@asenajs/asena-auth';

interface ChatData {
  authSession: AuthSession;
}

@WebSocket({ path: '/chat', name: 'ChatSocket', middlewares: [AuthWebSocketMiddleware] })
export class ChatSocket extends AsenaWebSocketService<ChatData> {
  protected async onOpen(ws: Socket<ChatData>): Promise<void> {
    ws.send(`Welcome, ${ws.data.values.authSession.user.id}`);
  }
}
```

Browsers cannot set headers on a WebSocket, so they send the token as the subprotocol pair `bearer, <token>`:

```typescript
const socket = new WebSocket('wss://api.example.com/chat', ['bearer', token]);
```

- The socket opens with `bearer` as its negotiated protocol. Server-side clients may send `Authorization: Bearer` instead; a same-origin page may rely on the cookie.
- Without a session the upgrade is answered 401 with `WWW-Authenticate: Bearer`; the socket never opens. A cross-site upgrade carrying only the cookie never opens.
- A subprotocol token wins over a cookie session resolved by a global `AuthSessionMiddleware`, and an invalid subprotocol token is refused even when the cookie is valid.

## Better Auth

Better Auth owns sign-up, sign-in, OAuth and the rest; the package puts its sessions behind the same guards. Extend `BetterAuthService` and hand `@BetterAuth` your instance:

```typescript
import { BetterAuth, BetterAuthService } from '@asenajs/asena-auth';
import { betterAuth } from 'better-auth';
import { bearer } from 'better-auth/plugins';

@BetterAuth(() => ({
  auth: betterAuth({
    // database, secret, social providers: your Better Auth configuration
    emailAndPassword: { enabled: true },
    plugins: [bearer()],
  }),
}))
export class AppBetterAuth extends BetterAuthService {}
```

Mount its handler with a controller whose prefix equals Better Auth's `basePath` (default `/api/auth`):

```typescript
import { Controller } from '@asenajs/asena/decorators';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { BetterAuthController } from '@asenajs/asena-auth';
import { AppBetterAuth } from '../auth/AppBetterAuth';

@Controller('/api/auth')
export class AuthApiController extends BetterAuthController {
  @Inject(AppBetterAuth)
  protected readonly authService!: AppBetterAuth;
}
```

- Every request below the prefix, any method, reaches Better Auth as the raw `Request`; its `Response` is returned unchanged. From there `@Protected`, `@Roles`, the session readers and both middlewares work as with the native provider.
- The thunk runs when the provider is constructed, not at import. It registers under the class name; the object form takes `name`. Other options: `primary`, `roles`.
- `Authorization: Bearer` and the WebSocket subprotocol token need Better Auth's `bearer` plugin; without it only Better Auth's cookie authenticates.
- Session lifetime, cookies and CSRF are Better Auth's configuration — the native options (`sessionTtl`, `cookie`, `csrf`, …) do not apply. Refreshed-session `Set-Cookie` headers are forwarded on the response.

## Multiple providers

```typescript
import { Protected } from '@asenajs/asena/auth';
import { Controller } from '@asenajs/asena/decorators';
import { Get } from '@asenajs/asena/decorators/http';
import { AsenaAuthService, Auth } from '@asenajs/asena-auth';
import type { Context } from '@asenajs/ergenecore';

@Auth({ primary: true })
export class AppAuth extends AsenaAuthService {}

@Auth({ name: 'Partner', cookie: { name: 'partner_session' } })
export class PartnerAuth extends AsenaAuthService {}

@Protected({ provider: 'Partner' })
@Controller('/partner')
export class PartnerController {
  @Get('/orders')
  orders(context: Context) {
    return context.send({ orders: [] });
  }
}
```

- A provider is known by its registered name: the class name, or `name` in the object form.
- Routes without a provider name, and both session middlewares, use the `primary: true` provider — or the only one.
- `@Protected({ provider })` asks only that provider; a session from any other provider is a 401 there.
- Give each native provider its own cookie name, or they share one cookie.
- Any class implementing core's `AsenaAuthProvider` (`getSession(request)`, `getRoles(session)`) registered with `@Service('ApiKeyAuth') @Implements(AUTH_PROVIDER_KEY)` joins the registry the same way (e.g. API keys for machine clients).

The registry checks every protected route at boot and fails it (`@OnStart hook 'AuthProviderRegistry.onStart()' failed`) when: several providers exist, none primary, and a route names none; a route names an unregistered provider; two providers are primary; a route is protected but no provider exists.

With several providers and none primary, `authComponents` cannot be imported whole — the two session middlewares need one provider, and the boot says so. Import `[AuthGuard, AuthGuardMiddleware, AuthProviderRegistry]` and subclass the middleware you need with its provider:

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { AuthSessionMiddleware } from '@asenajs/asena-auth';

@Middleware()
export class PartnerSessionMiddleware extends AuthSessionMiddleware {
  protected providerName = 'Partner';
}
```

`AuthWebSocketMiddleware` takes `providerName` the same way.

## Options

`@Auth(options)` or `@Auth(() => options)` — the thunk runs when the provider is constructed, so it can read the environment; it registers under the class name.

| Option | Type | Default | Notes |
|---|---|---|---|
| `name` | `string` | class name | Registered name for `@Protected({ provider })`; object form only |
| `primary` | `boolean` | `false` | Provider for routes and middlewares that name none |
| `sessionTtl` | `number` (s) | `604800` (7 days) | Session lifetime; also the cookie's `Max-Age` |
| `absoluteTtl` | `number \| false` (s) | `2592000` (30 days) | Hard cap from creation; sliding never passes it. Smaller than `sessionTtl` fails the boot |
| `sliding` | `boolean` | `true` | Extend once less than half of the lifetime is left |
| `cookie` | `CookieOptions` | see below | The session cookie |
| `bearer` | `boolean` | `true` | Accept `Authorization: Bearer`; `false` also disables the WebSocket subprotocol token |
| `csrf` | `{ trustedOrigins?: string[] } \| false` | on | Cross-site check for cookie sessions |
| `loadUser` | `(userId) => Promise<TUser \| null>` | — | Reload the user on every authenticated request |
| `roles` | `(session) => string[]` | `user.roles`, else comma-split `user.role` | Roles resolver for `@Roles` |
| `store` | `AuthSessionStore` | `MemoryAuthStore` | Where sessions live |
| `sweepInterval` | `number` (s) | `60` | How often the default `MemoryAuthStore` drops expired rows |
| `tokenBytes` | `number` | `32` | Random bytes per token |
| `hashing` | `{ algorithm?, cost? }` | bcrypt, cost 12 | `'bcrypt'`, `'argon2id'`, `'argon2d'`, `'argon2i'`; `cost` is bcrypt only |

Cookie defaults: `name: 'asena_session'`, `path: '/'`, `secure: true`, `sameSite: 'lax'`, no `domain` (host-only); `HttpOnly` is always set.

```typescript
import { AsenaAuthService, Auth } from '@asenajs/asena-auth';
import { findUserById } from '../db/users';

@Auth(() => ({
  sessionTtl: Number(process.env['SESSION_TTL'] ?? 86_400),
  csrf: { trustedOrigins: ['https://app.example.com'] },
  loadUser: (userId) => findUserById(userId),
}))
export class AppAuth extends AsenaAuthService {}
```

**`loadUser`.** By default a session carries the user snapshot from `createSession`, so a role change applies from the next login. With `loadUser`, its result replaces the stored user for that request and roles resolve from it — a demotion applies on the next request; `null`/`undefined` ends the session and deletes its row. `listUserSessions` and `rotateSession` without `user` keep the stored snapshot.

**Cross-site check (`csrf`).** A state-changing request (not `GET`/`HEAD`/`OPTIONS`) or a WebSocket upgrade carrying its token in the cookie and coming cross-site is treated as having no cookie — a 401 on a protected route. Cross-site = `Sec-Fetch-Site` other than `same-origin`/`none`; without it, an `Origin` different from the request URL's origin; neither header = non-browser client, passes. A same-site subdomain is not same-origin — list it in `trustedOrigins`. Bearer tokens are exempt; `csrf: false` disables the check.

## Session stores

`MemoryAuthStore` is in-process: gone on restart, not shared between instances. For more than one instance, implement `AuthSessionStore` over your database or Redis: `create(session)`, `get(id)`, `delete(id)`, `deleteByUser(userId)`, `touch(id, expiresAt)`, optional `listByUser(userId)` (without it `listUserSessions` throws), optional `onStart()`/`onStop()` (run once per store object even when providers share it). A `StoredSession` is `{ id: sha256(token) hex, userId, user, createdAt, expiresAt }` with epoch-millisecond times; `get` may return expired rows — the provider checks expiry.

Pass a store with `@Auth({ store })`. A store that needs injected services is set from the provider's own `onStart`, **before** `super.onStart()`:

```typescript
import { Inject } from '@asenajs/asena/decorators/ioc';
import { AsenaAuthService, Auth } from '@asenajs/asena-auth';
import { SessionRepository } from '../repositories/SessionRepository';
import { SqlSessionStore } from './SqlSessionStore';

@Auth({ sessionTtl: 3600 })
export class AppAuth extends AsenaAuthService {
  @Inject(SessionRepository)
  private sessions!: SessionRepository;

  public override async onStart(): Promise<void> {
    this.setStore(new SqlSessionStore(this.sessions));
    await super.onStart();
  }
}
```

## Testing

`@asenajs/asena-auth/testing` ships a provider that turns a request header into a session — no login step:

```typescript
import { describe, test } from 'bun:test';
import { createWebTest, silentLogger } from '@asenajs/asena/test';
import { createErgenecoreAdapter } from '@asenajs/ergenecore';
import { authComponents } from '@asenajs/asena-auth';
import { TestAuth, TestAuthProvider } from '@asenajs/asena-auth/testing';
import { ProjectController } from '../src/controllers/ProjectController';

@TestAuth({
  users: {
    ada: { id: 'u1', roles: ['admin'] },
    bob: { id: 'u2', roles: ['user'] },
  },
})
class AppTestAuth extends TestAuthProvider {}

describe('ProjectController', () => {
  test('guards by role', async () => {
    const { app } = await createWebTest({
      adapter: createErgenecoreAdapter({ logger: silentLogger }),
      controllers: [ProjectController],
      components: [...authComponents, AppTestAuth],
    });

    try {
      await app.get('/projects').expectStatus(401);
      await app.get('/projects', { headers: { 'x-test-user': 'bob' } }).expectStatus(200);
      await app.delete('/projects/1', { headers: { 'x-test-user': 'bob' } }).expectStatus(403);
      await app.delete('/projects/1', { headers: { 'x-test-user': 'ada' } }).expectStatus(200);
    } finally {
      await app.stop();
    }
  });
});
```

- The `x-test-user` header names a key of `users`; a missing header or unknown key is no session. `users` is read on every request, so a test may change it between requests. `@TestAuth` also takes `header`, `primary` and `name`, so it can stand in for one provider among several. Send the header on a WebSocket upgrade to drive `AuthWebSocketMiddleware`.
- `createWebTest` has no `imports` — the package components go into `components`. `createTestApp` takes them through `imports: [...authComponents]`.
- `TestAuthProvider` trusts a request header: it exists only under `/testing` — never register it in an application.

To test a real provider, resolve it and mint a session:

```typescript
await using app = await createTestApp({
  adapter: createErgenecoreAdapter({ logger: silentLogger }),
  components: [AppAuth, MeController],
  imports: [...authComponents],
});
const auth = await app.resolve<AppAuth>('AppAuth');
const { token } = await auth.createSession({ id: 'u1', roles: ['user'] });

await app.get('/me', { headers: { authorization: `Bearer ${token}` } }).expectStatus(200);
```

## Pitfalls

- **Marks without the guard protect nothing.** `@Protected` / `@Roles` only record metadata; without `AuthGuard` in `imports` every route stays public. `asena doctor`'s `auth-enforcement` check fails when `Protected`/`Roles` is imported from `@asenajs/asena/auth` but neither `authComponents` nor `AuthGuard` is imported from `@asenajs/asena-auth` — a static source scan, so it cannot tell whether the import actually reaches `AsenaServerFactory.create({ imports })`.
- **`@Protected` on a `@WebSocket` class (or a service) is ignored**, with a boot warning. Authenticate sockets with `AuthWebSocketMiddleware` in `@WebSocket({ middlewares })`.
- **Write the session cookie only through the helpers.** In a request whose session slid, the re-issued cookie is written after the handler. `applySessionCookie`, `clearSessionCookie`, `logout` and `rotateSession` cancel it; a bare `context.setCookie('asena_session', …)` is overwritten by it.
- **Durations are seconds**, not milliseconds — `sessionTtl: 3600` is one hour. `absoluteTtl` smaller than `sessionTtl` fails the boot.
- **`loadUser` has no DI access.** It is an option evaluated outside the provider instance — use a module-level function (as above), never `this.<injected field>`.
- **Unknown guard marks fail the boot.** A mark this version cannot enforce stops the boot instead of leaving the route open; a protected request whose route the guard cannot identify is a 500, never a pass. The guard needs the adapter's `ctx.routePattern` — both official adapters report it.
- **Guard before validation.** An anonymous request with an invalid body gets 401, not 400; a missing role is 403 without `WWW-Authenticate`. Route middlewares never see anonymous requests to protected routes.
- **Behind a TLS-terminating proxy** the app sees `http://` request URLs while browsers send an `https://` `Origin`. Browsers sending `Sec-Fetch-Site` pass on that alone; for the others list the public `https://` origin in `csrf.trustedOrigins` — `X-Forwarded-*` headers are not read.
- **`bearer: false` also disables the WebSocket subprotocol token.** A socket speaking its own subprotocol (e.g. `graphql-ws`) cannot carry the token — the token must be the first two entries and the socket negotiates `bearer`; authenticate it by cookie.
- **Better Auth without the `bearer` plugin** authenticates by cookie only — Bearer headers and WebSocket subprotocol tokens are ignored.
- **`MemoryAuthStore` is per process.** Several instances need a shared `AuthSessionStore`.
- **Several providers, none primary** → `@Protected()` without a provider name and a whole `authComponents` import both fail the boot. Mark one `primary: true`, or name providers and import the guard components individually.
- **Overriding `onStart` on a provider** must `await super.onStart()` — the base method carries `@OnStart` and validates options and starts the store.
- **One interface per provider class.** `@Auth` / `@BetterAuth` / `@TestAuth` register the class under `AUTH_PROVIDER_KEY`; it cannot also `@Implements` another interface.
- **`createSession` stores a snapshot** — keep the user small and plain; functions throw.

Deeper detail: `https://asena.sh/raw/packages/auth.md`.
