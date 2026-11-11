# Asena CLI

Command and configuration reference for `@asenajs/asena-cli` (as of 0.13.x): project scaffolding, code generation, dev runs, the browser console, diagnostics, and production bundling. Requires Bun >= 1.4.0; has no `@asenajs/asena` dependency — it reads the project's own core.

## Contents

- Command reference
- asena create
- asena generate — emitted shapes
- asena dev start
- asena console
- asena build
- asena init
- asena doctor
- asena-config.ts (defineConfig)
- buildOptions reference
- Suffix configuration (.asena/config.json)

## Command reference

```bash
bun install -g @asenajs/asena-cli   # install; verify with: asena --version
```

| Command | Shortcut | Description |
|---|---|---|
| `asena create [name]` | — | Create a new project (`.` = current directory) |
| `asena generate` | `asena g` | Generate project components |
| `asena generate controller` | `asena g c` | Generate a controller |
| `asena generate service` | `asena g s` | Generate a service |
| `asena generate middleware` | `asena g m` | Generate a middleware |
| `asena generate validator` | `asena g v` | Generate a Zod validator |
| `asena generate config` | `asena g config` | Generate a server config |
| `asena generate exception-handler` | `asena g eh` | Generate an `@ExceptionHandler` class |
| `asena generate websocket` | `asena g ws` | Generate a WebSocket namespace |
| `asena dev start` | — | Build once, then run (development only; slated for removal — prefer `bun run --hot src/index.ts`) |
| `asena console` | — | Run the app in development with a browser console for inspecting and driving it (needs core 0.13+) |
| `asena build` | — | Bundle for production |
| `asena init` | — | Create `asena-config.ts` in an existing project |
| `asena doctor` | — | Read-only check for common configuration mistakes (`--json` for machine output) |
| `asena --version` | `asena -V` | Show CLI version |
| `asena --help` | `asena -h` | Show help |

## asena create

Interactive without arguments; pass flags for SSH/CI (non-TTY prompts hang — always use flags there).

```bash
asena create my-project --adapter=hono --logger --eslint --prettier
asena create . --adapter=ergenecore --no-logger --no-eslint --no-prettier
asena create my-app --adapter=hono   # prompts for the remaining options
```

| Option | Values | Default |
|---|---|---|
| `[project-name]` | any string, `.` for cwd | prompted |
| `--adapter <adapter>` | `hono`, `ergenecore` | prompted |
| `--logger` / `--no-logger` | boolean | `true` |
| `--eslint` / `--no-eslint` | boolean | `true` |
| `--prettier` / `--no-prettier` | boolean | `true` |

Emits: `src/controllers/AsenaController.ts`, `src/index.ts`, `asena-config.ts`, `.asena/config.json` (+ schema), `package.json`, `tsconfig.json`, `.gitignore`; `src/logger/logger.ts` when logger enabled; ESLint/Prettier configs when enabled.

**Note:** `services/`, `middlewares/`, `config/`, `exceptions/`, `namespaces/` are NOT pre-created — `asena generate` creates each folder on first use, and the component scanner walks the entire `sourceFolder` tree, so layout is convention only.

## asena generate — emitted shapes

Generation is adapter-aware: imports and base classes come from the adapter recorded in `.asena/config.json` (`ergenecore` shown; `hono` swaps the adapter package). Each command prompts for a name; the suffix (below) is appended automatically.

```typescript
// asena g c User -> src/controllers/UserController.ts (body intentionally empty - add path + routes)
import { Controller } from '@asenajs/asena/decorators';
import { Get } from '@asenajs/asena/decorators/http';
import type { Context } from '@asenajs/ergenecore';

@Controller()
export class UserController {}
```

```typescript
// asena g s User -> src/services/UserService.ts
import { Service } from '@asenajs/asena/decorators';

@Service()
export class UserService {}
```

```typescript
// asena g m Auth -> src/middlewares/AuthMiddleware.ts (handle() is a placeholder stub - replace it)
import { Middleware } from '@asenajs/asena/decorators';
import { MiddlewareService, type Context } from '@asenajs/ergenecore';

@Middleware()
export class AuthMiddleware extends MiddlewareService {
  public async handle(context: Context, next: () => Promise<void>) {
    context.setValue('testValue', 'test');
    await next();
  }
}
```

```typescript
// asena g config Server -> src/config/ServerConfig.ts
import { Config } from '@asenajs/asena/decorators';
import { ConfigService } from '@asenajs/ergenecore';

@Config()
export class ServerConfig extends ConfigService {}
```

The config is empty: add `serveOptions()`, `globalMiddlewares()` or `transport()` as needed. Error handling does not go here — generate an exception handler.

```typescript
// asena g eh App -> src/exceptions/AppExceptionHandler.ts
import { ExceptionHandler } from '@asenajs/asena/decorators';
import { Catch, type ExceptionContext } from '@asenajs/asena/exception';
import { isHttpException } from '@asenajs/asena/adapter';
import type { Context } from '@asenajs/ergenecore';

@ExceptionHandler()
export class AppExceptionHandler {
  @Catch()
  public fallback(error: unknown, ctx: ExceptionContext<Context>) {
    if (ctx.scope !== 'http' || isHttpException(error)) return;

    console.error('Unhandled error:', error);

    return ctx.context.send({ error: 'Internal Server Error' }, 500);
  }
}
```

The generated catch-all needs core 0.13. It declines everything outside HTTP (a message caller still gets `REMOTE_ERROR`, events keep their retry / DLQ) and every `HttpException` — validation errors included — so those keep their own status and body; anything else is logged and answered with a 500 that does not leak the message. Add `@Catch(...)` methods for your own errors and swap `console.error` for an injected logger (`configuration-and-errors.md`).

```typescript
// asena g ws Chat -> src/namespaces/ChatNamespace.ts
// Adapter-agnostic: base class and Socket come from core, not the adapter package
import { WebSocket } from '@asenajs/asena/decorators';
import { AsenaWebSocketService, type Socket } from '@asenajs/asena/web-socket';

@WebSocket({ path: '/chat', name: 'ChatNamespace' })
export class ChatNamespace extends AsenaWebSocketService {
  protected async onOpen(ws: Socket): Promise<void> {}
  protected async onMessage(ws: Socket, message: string): Promise<void> {}
  protected async onClose(ws: Socket): Promise<void> {}
}
```

`asena g v` emits a Zod validator (adapter-specific base class; the docs do not publish its scaffold body).

## asena dev start

Builds the project, registers all discovered components, runs the bundle. Takes **no options** and does **not** watch for changes — for iteration use the scaffolded `bun run dev:hot` (`bun run --hot src/index.ts`). Not for production: deploy the `asena build` output instead.

## asena console

Runs the app as a child process and serves a browser UI that shows how it is wired and what it is doing, and can call it. Run it from the project root. Nothing goes in the project's code or config: no decorator, no `asena-config.ts` key, no extra package. The project needs `@asenajs/asena` >= 0.13.0, whose `@asenajs/asena/devtools` subpath holds the in-app agent, and an `asena-config.ts` for `sourceFolder` and `rootFile`.

```bash
asena console                            # prints the URL and opens the browser
asena console --port 4000 --no-open      # fixed console port, URL only
asena console --no-watch                 # no restart on save
```

| Option | Default | Effect |
|---|---|---|
| `--port <port>` | a free port | The console's own port (0–65535). The app keeps the port its own config sets |
| `--no-open` | opens the browser | Only print the URL |
| `--no-watch` | watches `sourceFolder` | Do not restart the app when a source file changes |

How a run works:

- **The entry file is never rewritten.** The CLI scans the components in a fresh Bun process, writes a temporary wrapper entry in the OS temp dir (as `asena build` does, plus the agent hook), and runs it from the project root with the project's environment and `tsconfig.json`. The app runs from source, not from a bundle.
- **A change restarts the whole process**; this is not hot reload, so in-memory state is lost. A change under `sourceFolder` triggers it (dot directories and editor temp files do not); so does the Restart button. The old process gets `SIGTERM`, then `SIGKILL` after 3 s.
- **A failed start or a crash leaves the console up.** The error appears in the Runtime output; the next save or Restart tries again.
- **Output goes to both places:** the terminal and the Runtime panel. `Ctrl+C` stops the console and the app.
- **Outside the console nothing runs.** Only the wrapper imports the agent, and core attaches it only when that hook is set in a process with an IPC channel; under `bun run`, `asena dev start` or an `asena build` bundle, request recording stays disabled and wraps no handler.
- **An app that runs with `NODE_ENV=production` reports nothing**, even under the console: core does not attach the agent or record anything, and logs one warning (`devtools agent not attached: NODE_ENV is production; the console is a development tool`). The app itself runs normally.
- **The agent reports once `AsenaServer.start()` finishes.** When it has not reported after 10 s the console says so in the terminal and the browser (`The app's devtools agent has not reported yet…`) and shows only the app's output until it does; it cannot tell a slow start from an app under `NODE_ENV=production`.
- **Every server of the process is reported.** Each `AsenaServer` the app starts attaches under its own server id, unique in the process and fixed for the server's life; its snapshot, its records and the results of its commands carry that id. With one server the console shows no picker and no server labels. With several, see "Several servers" below. While a server is stopping, a command is refused (`The server is stopping; the command was not run`) instead of run.
- **Output is captured per record.** `console.*` and `process.stdout` / `process.stderr` writes made while a request or dispatch is in scope are attached to it, console calls as the rendered text; every write still reaches the terminal. The agent's module loads before any module of the app, so a console wrapper the app installs (one that redacts secrets, say) sits on top of the capture and the record holds what that wrapper prints. A console method replaced before the agent's module loaded is not recorded through that replacement.
- **Telemetry is dropped before the app is slowed.** When the console is not reading fast enough, request records and metric samples are dropped and the metrics carry the count (`dropped`); commands and their answers are never dropped.
- **Session data lives in the console process**, capped (last 2000 log lines, 500 requests, 120 metric samples), and is replayed when a browser reconnects. It is gone when the console stops. A headless app works too: everything but HTTP and WebSocket testing.

The seven panels:

| Panel | Shows | Can do |
|---|---|---|
| Components | Graph of who injects whom and which middleware or validator guards what, with an inspector for the selected component | Focus or Full view (below), filter by kind, search |
| Runtime | App state, restarts this session and what caused each, memory, heap, CPU, event-loop lag, and the app's output | Filter by level, search, pause, clear |
| Endpoints | Every route and WebSocket namespace, with its middleware, validator and contract; path params come from the route and a route with a body schema starts from `{}` | Send a request and see the answer and chain, per-route history (kept in the browser); connect to a namespace and send text or JSON |
| Requests | Each HTTP request the app served, from anywhere, newest first, and each `@Schedule`, `@On` or message dispatch: status, duration, the chain of steps and what answered, and what was written to the console during it (first 100 writes or 16 KiB). Bodies and headers are not recorded | Filter by status, method and text; replay an HTTP request in Endpoints |
| Messaging | Schedules with cron and next run, events with their `@On` handlers, message handlers per transport with its connection state | Run now, Emit (JSON payload), Send (JSON payload; a `@MessagePattern` waits for the reply) |
| Config | Each `@Value` field with its key and whether it is set, `@Config`, adapter, port, core version, pid, global middleware in order, transports | — |
| Exceptions | Which error type goes to which `@Catch`, what answers when nothing matches, what answers an unmatched route | Type a class name to see who catches it |

Focus and Full (Components):

- **Default:** Focus when the app has more than 40 components, Full otherwise. A manual choice holds until the app crosses that limit or the page reloads.
- **Focus** shows the selected component and its direct neighbours, in both directions. Clicking a neighbour moves the focus. More than 3 validators or core services collapse into one group node; clicking it expands the group. Search finds any component, and choosing a result moves the focus.
- **Full** shows every component. On an app above the limit, core services and validators start hidden through the kind chips, nothing starts selected, the first fit follows the width with a 35% floor and is panned from there, and edges stay faint until a node is hovered or selected. A smaller app opens with its first controller selected.

Several servers (an app that starts more than one `AsenaServer` in its process):

- **A picker in the sidebar**, shown only then, names each server by adapter and port, and a headless one by its server id. It chooses whose components, endpoints, schedules, events, transports, config and exception handlers are on screen; the status pill and readiness follow it. Until one is picked, the most recently attached server that has not stopped is shown.
- **Actions go to the picked server.** Run now, Emit, Send and the endpoint tester's requests carry its server id, and each command result says which server answered.
- **Requests lists every server's records**, with a filter by server; an open record names its server, and replaying it in Endpoints needs that server picked.
- **A server that stops while the process runs on** stays in the picker, marked `(stopped)`, and takes no actions; the other servers keep reporting and answering.

The WebSocket tester connects straight from the browser to the app, so it can send no custom headers or subprotocol: a namespace behind token auth rejects it.

Security model — a development tool on a trusted machine, never a Dockerfile or start-script command:

- **Refuses `NODE_ENV=production`** with exit code 1.
- **Binds `127.0.0.1` only.** On a remote machine use `--no-open` and an SSH tunnel; do not expose the port. This covers the console's own server: the app still binds wherever its config says.
- **Tokened URL.** Each run generates a random token, carried in the URL's `t` parameter. Without it the page, the WebSocket and the request proxy answer `403`; only the static UI assets are open. Treat the printed URL as a credential.
- **Inline confirm on Run now, Emit and Send.** They run through the app's real dispatch paths, with real side effects. Endpoints' Send has no confirm and hits the real routes too, so point the app at a throwaway database.
- **`@Value` values stay in the app.** Config reports only whether each variable is set and tags credential-looking keys.

| Symptom | Cause / fix |
|---|---|
| `asena console needs @asenajs/asena 0.13.0 or newer…` | Upgrade core (the message names the installed version) |
| `asena console is a development tool and does not start when NODE_ENV=production.` | Unset `NODE_ENV` for this command |
| `No config file detected or invalid format` | No `asena-config.ts` below the working directory — run `asena init` or start from the project root |
| `[asena console] The app could not be started: …` | The component scan or the spawn failed (a syntax error, for instance); fix it and save |
| `[asena console] The app's devtools agent speaks protocol v…` | CLI and core speak different versions and the app's messages are dropped; update them to match |
| The app's output shows `devtools agent not attached: NODE_ENV is production…` and the panels have no app data | The app's process runs with `NODE_ENV=production` (set in code or a script, for instance); remove it for development |
| `[asena console] The app's devtools agent has not reported yet (10 s)…` | `AsenaServer.start()` has not finished (a slow or hanging `@OnStart`, a transport that will not connect), or the app runs with `NODE_ENV=production`; the panels fill in when the agent reports |
| Requests lists no HTTP rows | The adapter has no `attachDevtools()`: the app runs normally and only dispatches are recorded. Install the adapter release that pairs with core 0.13 |

## asena build

1. Reads `asena-config.ts`
2. Scans `sourceFolder` for controllers, services, middlewares, configs, exception handlers, websockets
3. Writes a **temporary wrapper entry** in the OS temp dir (outside `sourceFolder`): it imports every scanned component, publishes them on `globalThis[Symbol.for('asena.buildComponents')]`, then imports your entry file
4. Bundles that wrapper with Bun's bundler and deletes the temp files
5. Writes to `buildOptions.outdir` — output file is always `<outdir>/index.asena.js`

```bash
asena build
bun dist/index.asena.js   # run the production bundle
```

- **Your entry file is never rewritten** and its module-level code does not run at build time. Earlier CLI versions parsed the `AsenaServerFactory.create({...})` call and injected a `components: [...]` array, which imposed invisible formatting rules on the entry.
- **A hand-written `components: [...]` array is user-owned.** It is left untouched and, when non-empty, wins over the build's list. Delete it to let the build supply the components.
- **Requires the app's `@asenajs/asena` >= 0.11.0** (0.12.x and 0.13.x included) — 0.11.0 is the first core version that reads the build list. On an older core the bundle falls back to the filesystem scan and dies in production with `No components or configuration found`.
- **`minify.identifiers` is forced to `false`** (and `minify: true` is narrowed to `{ whitespace, syntax, identifiers: false }`), logging `[build] minify.identifiers disabled: component names are read at runtime`.

## asena doctor

Read-only diagnostics for the current project — reports, never modifies. Exit code `1` when any check fails, `0` otherwise, so it fits a CI step.

```bash
asena doctor          # ✓/✗ per check, with an indented `hint:` on failure
asena doctor --json   # the result array as JSON
```

| Check | Verifies |
|---|---|
| `tsconfig-decorators` | `experimentalDecorators` and `emitDecoratorMetadata` are both `true` |
| `asena-config` | `asena-config.ts` found and importable, `rootFile`/`sourceFolder` exist on disk, `minify.identifiers` not enabled |
| `direct-dependencies` | `@asenajs/asena` and `reflect-metadata` are direct deps, plus `hono`/`zod` when the matching adapter is installed |
| `duplicate-packages` | `@asenajs/asena`, `hono`, `zod` each resolve to a single version under `node_modules` (incl. the `.bun` store) |
| `peer-ranges` | Every installed `@asenajs/*` package's peer range for `@asenajs/asena` is satisfied by the installed core |
| `auth-enforcement` | When `Protected`/`Roles` is imported from `@asenajs/asena/auth`, `authComponents` or `AuthGuard` is imported from `@asenajs/asena-auth` — otherwise the marks are metadata only and those routes are served open |
| `auth-better-auth-dep` | When `@BetterAuth`, `BetterAuthService` or `BetterAuthController` is used, `better-auth` is in `package.json` and installed |
| `auth-cookie` | Literal `@Auth` `cookie` options form a cookie browsers accept: no `__Host-` name with `domain`, a non-`/` `path` or `secure: false`; no `__Secure-` name with `secure: false`; no `sameSite: 'none'` with `secure: false` |

The three `auth-*` checks read source files as text (skipping `node_modules`, `dist` and comments) and pass when they do not apply. `auth-enforcement` sees the import, not whether it reaches `imports: [...]`; `auth-cookie` skips non-literal values.

Run it first when diagnosing the silent failures below — `duplicate-packages` in particular is the fastest way to rule the `instanceof`/`HttpException` trap in or out. Known false positive: with CLI 0.11.0 installed in a core 0.12 or 0.13 project, the CLI's own `@asenajs/asena` 0.11.x under `node_modules/@asenajs/asena-cli/node_modules/` fails this check although the app never loads it (CLI 0.11.1 and later have no core dependency); only a second copy at any other path is the real trap.

## asena init

Creates `asena-config.ts` with defaults in an existing project (unnecessary after `asena create`):

```typescript
import { defineConfig } from '@asenajs/asena-cli';

export default defineConfig({
  sourceFolder: 'src',
  rootFile: 'src/index.ts',
  buildOptions: {
    outdir: 'dist',
    minify: { whitespace: true, syntax: true, identifiers: false, keepNames: true },
  },
});
```

## asena-config.ts (defineConfig)

```typescript
import { defineConfig } from '@asenajs/asena-cli';

interface AsenaConfig {
  sourceFolder: string;         // component scan root
  rootFile: string;             // application entry point
  include?: string[];           // files/dirs copied into outdir at build
  buildOptions?: BuildOptions;  // subset of Bun.BuildConfig (below)
}
```

| Property | Type | Default | Notes |
|---|---|---|---|
| `sourceFolder` | `string` | `'src'` | Scanner discovers every decorated class anywhere under this tree |
| `rootFile` | `string` | `'src/index.ts'` | Kept out of the component sweep — importing an entry suspended on top-level `await` is a cyclic import that hangs startup. Components declared inside the entry still register if placed above the `AsenaServerFactory.create()` call |
| `include` | `string[]` | `[]` | Paths relative to project root, copied recursively into outdir, structure preserved. Required for `@FrontendController` HTML pages (import paths are rewritten automatically) |
| `buildOptions` | `BuildOptions` | `{ outdir: './out' }` when omitted | The scaffolded config sets `outdir: 'dist'` explicitly, which is why projects build into `dist/` |

## buildOptions reference

Only backend-relevant Bun bundler options are exposed; `entrypoints` and `target` are managed internally (always `bun` target). `splitting`, `define`, `loader` are not exposed.

| Option | Type | Default | Notes |
|---|---|---|---|
| `outdir` | `string` | `'dist'` (scaffold) | Compiled output directory |
| `sourcemap` | `'none' \| 'inline' \| 'external' \| 'linked'` | unset | `'linked'`/`'external'` for debugging, `'none'` for production |
| `minify` | `boolean \| MinifyOptions` | `{ whitespace: true, syntax: true, identifiers: false, keepNames: true }` (what `asena init` writes) | See below |
| `external` | `string[]` | `[]` | Excluded from the bundle, resolved at runtime — required for native-binding packages (e.g. `better-sqlite3`, `pg`, `mysql2`) |
| `format` | `'esm' \| 'cjs'` | `'esm'` | ESM recommended on Bun |
| `drop` | `string[]` | `[]` | Removes call expressions (`'console'`, `'debugger'`, `'logger.debug'`) |

Minify sub-options: `whitespace` (strip whitespace), `syntax` (condensation transforms), `identifiers` (rename to short names), `keepNames` (preserve function/class names).

- **Warning:** `identifiers` must stay `false`. Component registration is name-based, so renaming classes breaks `@Inject('UserService')` lookups at runtime — in production only. `asena build` forces it to `false` and `asena doctor` reports a config that enables it.
- **`keepNames: true` is NOT a safeguard.** Measured on Bun 1.4.0: `Bun.build` does not preserve class names under identifier minification whether `keepNames` sits inside `minify` or beside it. It is harmless and buys readable stack traces; `identifiers: false` is the only rule that works. Never recommend `keepNames` as the fix.
- **Warning:** `drop` removes the entire call expression including arguments — `drop: ['console']` turns `console.log(doSomething())` into nothing, so `doSomething()` never executes.

```typescript
// Production profile
export default defineConfig({
  sourceFolder: 'src',
  rootFile: 'src/index.ts',
  buildOptions: {
    outdir: 'dist',
    format: 'esm',
    sourcemap: 'none',
    minify: { whitespace: true, syntax: true, identifiers: false },
    drop: ['console', 'debugger'],
  },
});
```

## Suffix configuration (.asena/config.json)

`.asena/config.json` (created by `asena create` / `asena init`) stores the adapter and the class-name suffixes `asena generate` appends. Changes apply immediately, no restart. A missing `suffixes` field means defaults (`true`).

Defaults: Controller → `Controller`, Service → `Service`, Middleware → `Middleware`, Config → `Config`, Exception handler (`exceptionHandler` key) → `ExceptionHandler`, WebSocket → `Namespace`.

```json
{
  "adapter": "hono",
  "suffixes": {
    "controller": true,     // default suffix -> asena g c User = UserController.ts
    "service": "Svc",       // custom        -> asena g s Auth = AuthSvc.ts
    "middleware": false,    // none          -> asena g m Logger = Logger.ts
    "config": true,
    "websocket": "Socket"
  }
}
```

Per type: `true` = default suffix, `false` = no suffix, `"Custom"` = custom suffix, omitted = default. `"suffixes": true` / `false` sets all types at once. Suffixes must be valid TypeScript class-name segments (letters and numbers only) — use `false`, never `""`, to disable.
