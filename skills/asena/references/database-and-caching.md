# Database & Caching

Drizzle ORM integration (`@asenajs/asena-drizzle`) and Redis client + declarative caching (`@asenajs/asena-redis`).

## Contents
- Drizzle Setup
- @Database Options
- Repositories
- Transactions
- Multiple Databases
- Redis Client
- Caching Decorators

## Drizzle Setup

```bash
bun add @asenajs/asena-drizzle drizzle-orm
bun add pg       # for type: 'postgresql'
bun add mysql2   # for type: 'mysql'
# type: 'bun-sql' uses Bun's built-in SQL — no driver package
```

Requires Bun >= 1.3.12, `@asenajs/asena` >= 0.10.0, `drizzle-orm` >= 0.45.2. SQLite is not yet supported.

**Schema-export pattern** — aggregate all schemas into one default-export object; the generics below depend on it for inference:

```typescript
// database/schemas/user.schema.ts
import { pgTable, uuid, text, boolean } from 'drizzle-orm/pg-core';
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  isActive: boolean('is_active').default(true),
});

// database/schemas/index.ts
import * as User from './user.schema';
export default { ...User };
```

**Database service** — subclass `AsenaDatabaseService`, decorate with `@Database`:

```typescript
import { Database, AsenaDatabaseService } from '@asenajs/asena-drizzle';
import type { BunSQLDatabase } from 'drizzle-orm/bun-sql';
import Schemas from '../database/schemas';

@Database({
  type: 'bun-sql',
  config: { host: 'localhost', port: 5432, database: 'myapp', user: 'postgres', password: 'password' },
  name: 'MainDatabase', // IoC registration name — always set it
})
export class MyDatabase extends AsenaDatabaseService<BunSQLDatabase<typeof Schemas>> {}
```

Database type generic per driver (use it on both `AsenaDatabaseService<T>` and `BaseRepository<Table, T>`):

| `type` | Driver | Generic |
|---|---|---|
| `'postgresql'` | `pg` | `NodePgDatabase<typeof Schemas>` |
| `'mysql'` | `mysql2` | `MySql2Database<typeof Schemas>` |
| `'bun-sql'` | Bun native | `BunSQLDatabase<typeof Schemas>` |

The pool is opened by an inherited `@OnStart` and released by `@OnStop`, so `server.stop()` returns connections (needs core >= 0.10.0). Never manage the connection yourself.

## @Database Options

```typescript
@Database({
  type: 'postgresql' | 'mysql' | 'bun-sql',
  config: {
    host?: string; port?: number; database?: string; user?: string; password?: string;
    ssl?: boolean;
    connectionString?: string;      // replaces the five discrete fields — pick ONE style
    name?: string;                  // shown in the connection log
    pool?: {                        // driver-agnostic, all durations in ms
      max?: number; idleTimeoutMs?: number; connectTimeoutMs?: number; maxLifetimeMs?: number;
    };
    extra?: Record<string, unknown>; // driver-native options, spread LAST — wins over everything
  },
  name?: string;                    // IoC service name (recommended, required in practice for multiple DBs)
  logger?: ServerLogger;
  drizzleConfig?: { logger?: boolean; schema?: any; configPath?: string };
})
```

Pool mapping (undefined fields keep the driver default): `max` → pg `max` (20) / mysql2 `connectionLimit` (10) / bun `max` (10); `idleTimeoutMs` → pg `idleTimeoutMillis` (30000) / mysql2 `idleTimeout` / bun `idleTimeout` (converted to seconds); `connectTimeoutMs` → pg `connectionTimeoutMillis` (2000) / mysql2 `connectTimeout` / bun `connectionTimeout` (seconds); `maxLifetimeMs` → pg `maxLifetimeSeconds` / bun `maxLifetime` (seconds), ignored on mysql2. The shape is exported as `DatabasePoolConfig`.

**Connection string:**

```typescript
@Database({
  type: 'postgresql',
  config: { connectionString: process.env.DATABASE_URL, pool: { max: 20 } },
  name: 'MainDatabase',
})
export class DatabaseFromURL extends AsenaDatabaseService {}
```

- When `connectionString` is set, the discrete fields are **not sent to the driver at all**. `ssl`, `pool`, `extra` still apply.
- **Never supply both** URL and discrete fields — drivers disagree on precedence (pg lets the URL win and falls back to `PGHOST`/defaults for keys the URL omits; mysql2 keeps truthy discrete options and ignores the URI).
- **Warning (0.10.0):** before core 0.10.0 only `bun-sql` read `connectionString`; pg/mysql silently connected from discrete fields/env. Fixed — the five fields are now optional.
- `ssl` precedence differs: on pg an `ssl`/`sslmode` in the URL query string overrides `config.ssl` both ways; on mysql2/bun-sql an explicit `ssl: true` beats the URL.

## Repositories

```typescript
import { Repository, BaseRepository } from '@asenajs/asena-drizzle';
import { eq } from 'drizzle-orm';

@Repository({
  table: users,                 // Drizzle table — MUST have an 'id' column
  databaseService: 'MainDatabase',
  name?: 'UserRepository',      // optional, defaults to class name
})
export class UserRepository extends BaseRepository<typeof users, BunSQLDatabase<typeof Schemas>> {
  async findByEmail(email: string) {
    return this.findOne(eq(users.email, email));
  }
}
```

Always pass **both** generics (`typeof table`, database type) — without the second, `this.db` loses the query-builder types.

**Layering rule:** inject a repository ONLY into its own domain's service (`UserRepository` → `UserService`). Never inject it into a controller or another domain's service — cross-domain access goes through the owning service's methods.

### BaseRepository methods

| Method | Signature | Notes |
|---|---|---|
| `findById` | `(id) => row \| undefined` | |
| `findOne` | `(where) => row \| undefined` | Drizzle condition (`eq`, `and`, ...) |
| `findAll` | `(where?) => row[]` | No filter = all rows |
| `findBy` | `(field, value) => row[]` | Typed shortcut for `findAll(eq(table[field], value))` |
| `create` | `(data) => row` | |
| `createMany` | `(data[]) => row[]` | |
| `updateById` | `(id, data) => row` | |
| `update` / `updateMany` | `(where, data)` | `updateMany` is a semantic alias |
| `deleteById` | `(id)` | |
| `delete` / `deleteMany` | `(where)` | `deleteMany` returns removed count |
| `count` | `() => number` | |
| `countBy` | `(where) => number` | |
| `exists` | `(where) => boolean` | |
| `existsBy` | `(field, value) => boolean` | Typed shortcut for `exists(eq(...))` |
| `paginate` | `(page, limit, where?, orderBy?)` | Returns `{ data, total, page, limit, totalPages }`; defaults to `asc(table.id)` for stable pages |
| `transaction` | `(callback, options?)` | ALS-aware: SAVEPOINT inside an active `@Transaction`, else new top-level tx; `options` forwards `isolationLevel`/`accessMode` |
| `db` (getter) | raw Drizzle connection | Full query builder, joins, `db.execute(sql\`...\`)` |

All methods are async. Inject repositories into services with `@Inject('UserRepository')`.

## Transactions

`@Transaction` is Spring-style, propagated through `AsyncLocalStorage` — repository calls inside a wrapped method join the active transaction with no `tx` parameter.

**Setup (required):** subclass `TransactionPostProcessor` inside your source folder (AsenaJS scans `src`, never `node_modules`). Without this class `@Transaction` is **silently inert**.

```typescript
// src/config/AppDrizzle.ts
import { Drizzle, TransactionPostProcessor } from '@asenajs/asena-drizzle';

@Drizzle({ defaultDb: 'MainDatabase' }) // defaultDb optional; lets @Transaction() omit `database`
export class AppDrizzle extends TransactionPostProcessor {} // body intentionally empty
```

```typescript
import { Transaction } from '@asenajs/asena-drizzle';

@Service('AccountService')
export class AccountService {
  @Inject('UserRepository') private userRepo: UserRepository;
  @Inject('AuditRepository') private auditRepo: AuditRepository;

  @Transaction() // uses defaultDb; both rows commit or both roll back
  async register(payload: { email: string; name: string }) {
    const user = await this.userRepo.create(payload);
    await this.auditRepo.create({ userId: user.id, event: 'register' });
    return user;
  }
}
```

Options: `@Transaction({ database?, propagation?, isolationLevel?, accessMode? })`. `database` overrides `defaultDb`; if neither is set, registration fails fast with a descriptive error.

| `propagation` | Transaction active | No transaction |
|---|---|---|
| `REQUIRED` (default) | Joins it | Starts new top-level tx |
| `NESTED` | Opens a SAVEPOINT; inner failure rolls back to it without aborting the outer tx | Starts new top-level tx |
| `REQUIRES_NEW` | Suspends outer, runs independent top-level tx on a fresh pooled connection | Starts new top-level tx |

- `isolationLevel`: `'read uncommitted' | 'read committed' | 'repeatable read' | 'serializable'`; `accessMode`: `'read only' | 'read write'`. Forwarded to Drizzle's `db.transaction(cb, config)` only on top-level paths; a `NESTED` SAVEPOINT inherits the outer isolation. Unsupported values are silently ignored per dialect.
- **Never** use arrow-function class properties (`run = async () => ...`) — they live on the instance, not the prototype, and are not intercepted. Use `async method() {}`.
- Programmatic boundary: `repo.transaction(cb, opts)` (see table above). Full query builder inside a tx: `this.repo.db.transaction(async (tx) => { ... })`.

## Multiple Databases

```typescript
@Database({ type: 'postgresql', config: {/* ... */}, name: 'PrimaryDB' })
export class PrimaryDatabase extends AsenaDatabaseService {}

@Database({ type: 'mysql', config: {/* ... */}, name: 'AnalyticsDB' })
export class AnalyticsDatabase extends AsenaDatabaseService {}

@Repository({ table: users, databaseService: 'PrimaryDB' })
export class UserRepository extends BaseRepository<typeof users> {}

@Repository({ table: events, databaseService: 'AnalyticsDB' })
export class EventRepository extends BaseRepository<typeof events> {}
```

With multiple databases, every `@Transaction` must name its target: `@Transaction({ database: 'AnalyticsDB' })`.

## Redis Client

```bash
bun add @asenajs/asena-redis          # Bun-native RedisClient (default adapter)
bun add @asenajs/asena-redis redis    # only if using adapter: 'node-redis'
```

Requires Bun >= 1.3.12, `@asenajs/asena` >= 0.10.0. Zero runtime dependencies.

```typescript
import { Redis, AsenaRedisService } from '@asenajs/asena-redis';

@Redis({
  config: { url: 'redis://localhost:6379' },
  adapter?: 'bun' | 'node-redis',   // default 'bun'
  name: 'AppRedis',                 // IoC name — always set it
})
export class AppRedis extends AsenaRedisService {}
```

Inject with `@Inject('AppRedis')`. Connect/disconnect are automatic: `@OnStart` connects, `@OnStop` closes subscribers then the main client (a user-supplied `client` option is adopted and closed too).

`RedisConfig`: `url` (`redis[s]://[[user][:pass]@][host][:port][/db]`), or `host` (default `'localhost'`) / `port` (default `6379`) / `username` / `password` / `db`; `connectionTimeout` (ms, default 10000), `autoReconnect` (default true), `maxRetries` (default 10), `enableOfflineQueue` (default true), `tls?: boolean | TLSOptions`, `name`. Bun-adapter-only (ignored on node-redis): `idleTimeout` (default 0), `enableAutoPipelining` (default true).

### Operations

| Group | Methods |
|---|---|
| String | `get(key)` (returns `string \| null` — **`null` only on a miss**, cached falsy values are hits), `set(key, value, ttl?)` (ttl seconds), `del(...keys)`, `exists(key)`, `incr(key)`, `decr(key)`, `expire(key, seconds)`, `ttl(key)`, `keys(pattern)` |
| Hash | `hget(key, field)`, `hmset(key, ['f1','v1','f2','v2'])`, `hmget(key, fields)` |
| Set | `sadd(key, member)`, `srem(key, member)`, `smembers(key)`, `sismember(key, member)` |
| Raw/lifecycle | `send(command, args)`, `client` (underlying `RedisClientAdapter`), `createSubscriber()` (duplicate connection for pub/sub, auto-closed on stop), `testConnection()`, `disconnect()` (called for you by `@OnStop`) |

## Caching Decorators

**Setup (required):** subclass `CachePostProcessor` inside your source folder; without it the cache decorators only record metadata and are **silently inert** (methods work, Redis is never touched).

```typescript
// src/config/AppCache.ts
import { RedisCache, CachePostProcessor } from '@asenajs/asena-redis';

@RedisCache({ ttl: 300 }) // application-wide defaults for every cache decorator
export class AppCache extends CachePostProcessor {}
```

```typescript
import { Cacheable, CachePut, CacheEvict, CacheConfig } from '@asenajs/asena-redis';

@Service()
@CacheConfig({ prefix: 'user' })
export class UserService {
  @Cacheable({ key: (id: string) => id, ttl: 60 })
  async findById(id: string): Promise<User | null> { /* db read */ }

  @CachePut({ key: (user: User) => user.id })
  async refresh(user: User): Promise<User> { /* always runs, rewrites cache */ }

  @CacheEvict({ key: (id: string) => id })
  async update(id: string, dto: UpdateDto): Promise<void> { /* evicts after success */ }

  @CacheEvict({ allEntries: true })
  async purge(): Promise<void> { /* SCAN-deletes everything under the prefix */ }
}
```

### Semantics

| | Method runs | Reads cache | Writes cache | If method throws |
|---|---|---|---|---|
| `@Cacheable` | only on miss | before call | on miss, after success | no write, error propagates |
| `@CachePut` | always | never | after success | no write, error propagates |
| `@CacheEvict` | always | never | deletes **after** success | no eviction |
| `@CacheEvict({ beforeInvocation: true })` | always | never | deletes **before** call | eviction already happened |

Stacked decorators produce one wrapper with fixed order `evict(before) → read → method → write → evict(after)`. `@Cacheable` + `@CachePut` on the same method is rejected at boot.

### Options

Resolution per option, nearest wins: `method option > @CacheConfig (class) > @RedisCache (application)`.

| Option | Applies to | Meaning |
|---|---|---|
| `key` | all three | `(...args) => string`. Omitted → `` `${Class.name}.${method}:${JSON.stringify(args)}` `` |
| `prefix` | all three | Key namespace joined with `:`; also the scope of `allEntries` |
| `ttl` | `@Cacheable`, `@CachePut` | **Seconds.** Omitted = never expires |
| `redis` | all three | `@Redis` service name. Omit only when exactly one is registered |
| `cacheNull` | `@Cacheable`, `@CachePut` | Store `null`/`undefined` results (default `true`) |
| `allEntries` | `@CacheEvict` | SCAN-delete every key under the prefix; requires a prefix |
| `beforeInvocation` | `@CacheEvict` | Evict before the method (default `false`) |

Key rules: the default key embeds `Class.name`, so an inherited `@Cacheable` caches **per subclass**, and minification changes every derived key — always supply an explicit `key` for anything long-lived. Key-generator parameters are not type-checked against the method's.

### Serialization, failure, limits

- Values are stored as JSON: whatever survives `JSON.parse(JSON.stringify(x))`. `Date` → ISO string; `Map`/`Set` → `{}`; class instance → plain object; `undefined`/`null`/`0`/`''`/`false` preserved and distinguishable from a miss; circular/`BigInt` → returned to caller but the cache write is skipped with a warning.
- **Runtime Redis failures fail open** (warning logged, method runs uncached). **Configuration mistakes fail fast at boot** (decorator on a non-method, `@Cacheable`+`@CachePut`, `allEntries` without prefix, non-positive `ttl`); a throwing `key` function propagates.
- **Warning:** with more than one `@Redis` service and no `redis` option named, the error surfaces on the **first cached call**, not at boot — always name the service when multiple exist.
- Wrapped methods always return a Promise (sync methods become async). Arrow-function class properties are not supported. `allEntries` SCAN is not a snapshot and only covers one node on Redis Cluster. No stampede protection — concurrent misses all run the method.

Deeper detail: `https://asena.sh/raw/packages/drizzle.md`, `https://asena.sh/raw/packages/redis.md`.
