# Events & Scheduling

In-process event system (`@EventService`/`@On`/`emitter()`) and cron-based scheduled tasks (`@Schedule`/`AsenaSchedule`/`CronRunner`).

## Contents

- Event System
- @EventService and @On
- Emitting Events
- Event Semantics and Rules
- Scheduled Tasks
- CronRunner

## Event System

Fire-and-forget, in-process only: `emit()` returns immediately, handlers run in the background, handler errors are caught and logged without affecting other handlers or the emitter. Use it for domain events inside one app; for cross-service messaging with delivery guarantees use the microservice layer (`ulak.send/emit` + `@MessagePattern`/`@EventPattern`) — the two are deliberately separate APIs.

```typescript
import { EventService } from '@asenajs/asena/decorators';
import { On } from '@asenajs/asena/event';

@EventService({ prefix: 'user' })
export class UserEventService {
  @On('created')           // handles 'user.created'
  handleUserCreated(eventName: string, data: any) { /* ... */ }

  @On('*.error')           // handles 'user.<anything>.error'
  handleUserError(eventName: string, data: any) { /* ... */ }
}
```

Event services are full IoC components — `@Inject` works, including injecting `emitter()` to chain events. Discovery/registration happens automatically at bootstrap.

## @EventService and @On

`@EventService(params?)` — marks a class as an event handler container.

| Param | Type | Description |
|---|---|---|
| `prefix` | `string` | Prepended (dot-joined) to every `@On` pattern in the class |
| `name` | `string` | IoC component name (default: class name) |

String shorthand: `@EventService('order')` ≡ `@EventService({ prefix: 'order' })`.

`@On(params)` — marks a method as a handler. String shorthand for the pattern, or object:

| Param | Type | Description |
|---|---|---|
| `event` | `string` | Event pattern (exact or wildcard) |
| `prefix` | `boolean` | Prepend the class prefix (default `true`); `false` = absolute pattern |
| `skip` | `boolean` | Skip this handler (debugging) |

Handler signature: `(eventName: string, data?: any) => void | Promise<void>`. Return values are ignored.

**Warning:** One `@On` per method — metadata is keyed by method name, so stacked `@On` decorators silently keep only the topmost. Use several methods or a wildcard.

Prefix building:

| Prefix | Pattern | Final |
|---|---|---|
| `''` | `user.created` | `user.created` |
| `user` | `created` | `user.created` |
| `user` | `*` | `user.*` |
| `user` | `*.updated` | `user.*.updated` |
| `user` | `payment.done` + `prefix: false` | `payment.done` |

**Note:** Same prefix rule as microservice `@MessageController` (applies to both `@MessagePattern` and `@EventPattern`; `prefix: false` opts out).

### Wildcards

| Pattern | Matches |
|---|---|
| `*` | All events |
| `user.*` | All user events at **any** depth (`user.created`, `user.profile.updated`) |
| `*.error` | All error events at any depth |
| `user.*.created` | Nested (`user.admin.created`) |

**Warning:** `*` matches one **or more** segments, never zero — `download.*` does not match the bare event `download`. Wildcards are slower than exact matches; reserve for cross-cutting concerns (logging, monitoring, `*.error`).

## Emitting Events

```typescript
import { Service } from '@asenajs/asena/decorators';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { emitter } from '@asenajs/asena/event';
import type { EventEmitter } from '@asenajs/asena/event';

@Service()
export class UserService {
  @Inject(emitter())
  private emitter!: EventEmitter;

  async createUser(name: string) {
    const user = { id: 123, name };
    this.emitter.emit('user.created', user); // returns immediately
    return user;
  }
}
```

`emit(eventName: string, data?: any): boolean` — returns `true` if any handler matched, `false` otherwise. Use dot-notation names (`order.payment.completed`), not camelCase/underscores.

## Event Semantics and Rules

- **Fire-and-forget:** `emit()` never waits, even for async handlers. There is no way to await handler completion — by design.
- **Error isolation:** a throwing handler is caught and logged; other handlers and the emitter continue.
- **No execution order guarantee** across handlers of one event. For sequencing, chain events: handler 1 emits the event handler 2 listens on.
- **Keep handlers lightweight** — emit follow-up events (`email.send`, `report.generate`) instead of doing heavy work inline.
- Handlers can emit further events (chains); `@On` handlers inherited from a base class are registered (see inheritance docs).

## Scheduled Tasks

Decorate a class with `@Schedule` and implement `AsenaSchedule`; Asena discovers and registers it at bootstrap and runs `execute()` on the cron schedule.

```typescript
import { Schedule } from '@asenajs/asena/decorators';
import { Inject } from '@asenajs/asena/decorators/ioc';
import type { AsenaSchedule } from '@asenajs/asena/schedule';

@Schedule({ cron: '0 3 * * *' })  // daily 03:00
export class DatabaseCleanup implements AsenaSchedule {
  @Inject('SessionRepository')     // full IoC component - injection works
  private sessionRepo: SessionRepository;

  public async execute() {
    await this.sessionRepo.deleteExpired();
  }
}
```

`@Schedule` options: `cron` (required expression), `name` (optional IoC component name).

```typescript
export interface AsenaSchedule {
  execute(): Promise<void> | void;
}
```

If `execute()` throws, the framework catches and logs it; the schedule keeps running — one failure never stops future executions.

**Note:** Cron expressions are validated with Bun's native `Bun.cron.parse()` **inside the decorator, at module import** — an invalid expression throws before `AsenaServerFactory.create()` is reached, so the server cannot start misconfigured. Nicknames (`@hourly`, `@daily`, `@weekly`, `@monthly`, `@yearly`) are accepted.

### Cron Format

5-field format: `minute hour day-of-month month day-of-week`.

| Field | Values | Special chars |
|---|---|---|
| Minute | 0-59 | `*` `,` `-` `/` |
| Hour | 0-23 | `*` `,` `-` `/` |
| Day of Month | 1-31 | `*` `,` `-` `/` |
| Month | 1-12 | `*` `,` `-` `/` |
| Day of Week | 0-6 (Sun=0) or MON-SUN | `*` `,` `-` `/` |

| Expression | Meaning |
|---|---|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 9 * * MON-FRI` | Weekdays 09:00 |
| `0 0 1 * *` | First of the month, midnight |
| `0 0 * * 0` | Sundays, midnight |

## CronRunner

Core framework service managing all scheduled jobs. Inject via `ICoreServiceNames.CRON_RUNNER`:

```typescript
import { Inject } from '@asenajs/asena/decorators/ioc';
import { ICoreServiceNames } from '@asenajs/asena/ioc/types';
import type { CronRunner } from '@asenajs/asena/schedule';

@Inject(ICoreServiceNames.CRON_RUNNER)
private cronRunner: CronRunner;
```

| Member | Type | Description |
|---|---|---|
| `getJobNames()` | `string[]` | Registered job names |
| `registerJob(name, cron, fn)` | `void` | Register a job programmatically |
| `startAll()` / `stopAll()` | `void` | Start/stop every registered job |
| `clearJobs()` | `void` | Stop and drop all jobs |
| `jobCount` | `number` | Registered job count |
| `hasRunningJobs` | `boolean` | Any job **scheduled** (started, not stopped) — not "executing right now" |

Lifecycle is automatic: jobs start when the server is ready — **after** every `@OnStart` has returned — and stop during graceful shutdown **before** any `@OnStop` runs (first step of `server.stop()` is "take no new work"). A job already executing at shutdown is not cancelled; `stopAll()` only unschedules future runs.
