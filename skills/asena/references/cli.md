# Asena CLI

Command and configuration reference for `@asenajs/asena-cli` (as of 0.11.x): project scaffolding, code generation, dev runs, diagnostics, and production bundling. Requires Bun ≥ 1.3.12.

## Contents

- Command reference
- asena create
- asena generate — emitted shapes
- asena dev start
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
| `asena generate websocket` | `asena g ws` | Generate a WebSocket namespace |
| `asena dev start` | — | Build once, then run (development only; slated for removal — prefer `bun run --hot src/index.ts`) |
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

**Note:** `services/`, `middlewares/`, `config/`, `namespaces/` are NOT pre-created — `asena generate` creates each folder on first use, and the component scanner walks the entire `sourceFolder` tree, so layout is convention only.

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
import { ConfigService, type Context } from '@asenajs/ergenecore';

@Config()
export class ServerConfig extends ConfigService {
  public onError(error: Error, context: Context): Response | Promise<Response> {
    return context.send({ error: error.message }, 500);
  }
}
```

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

## asena build

1. Reads `asena-config.ts`
2. Scans `sourceFolder` for controllers, services, middlewares, configs, websockets
3. Writes a **temporary wrapper entry** in the OS temp dir (outside `sourceFolder`): it imports every scanned component, publishes them on `globalThis[Symbol.for('asena.buildComponents')]`, then imports your entry file
4. Bundles that wrapper with Bun's bundler and deletes the temp files
5. Writes to `buildOptions.outdir` — output file is always `<outdir>/index.asena.js`

```bash
asena build
bun dist/index.asena.js   # run the production bundle
```

- **Your entry file is never rewritten** and its module-level code does not run at build time. Earlier CLI versions parsed the `AsenaServerFactory.create({...})` call and injected a `components: [...]` array, which imposed invisible formatting rules on the entry.
- **A hand-written `components: [...]` array is user-owned.** It is left untouched and, when non-empty, wins over the build's list. Delete it to let the build supply the components.
- **Requires `@asenajs/asena` ^0.11.0** — the first core version that reads the build list. On an older core the bundle falls back to the filesystem scan and dies in production with `No components or configuration found`.
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

Run it first when diagnosing the silent failures below — `duplicate-packages` in particular is the fastest way to rule the `instanceof`/`HttpException` trap in or out.

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

Defaults: Controller → `Controller`, Service → `Service`, Middleware → `Middleware`, Config → `Config`, WebSocket → `Namespace`.

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
