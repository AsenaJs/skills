# Microservices

Broker-agnostic service-to-service messaging: `@MessageController` handlers, the `ulak` client, Redis Streams and Kafka transports, delivery semantics, headless mode.

## Contents

- Message Controllers
- The Prefix Rule
- MessageContext
- Client API: ulak.messages
- Transport Configuration
- Redis Streams Transport
- Kafka Client (produce/consume without microservices)
- Kafka Transport
- Kafka External Topics (Interop)
- Delivery Semantics
- Idempotency and DLQ
- Multiple Named Transports
- Headless Mode and Health
- Graceful Shutdown
- InMemoryTransport (Testing)
- UlakError Codes
- Custom Transport SPI

## Message Controllers

`@MessagePattern` = request/response (RPC, return value is the reply). `@EventPattern` = fire-and-forget events (wildcards allowed).

```typescript
import { MessageController } from '@asenajs/asena/decorators';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { MessagePattern, EventPattern, type MessageContext } from '@asenajs/asena/microservice';

@MessageController('order') // prefix applied to every handler below
export class OrderHandler {
  @Inject('OrderService')
  private orderService: OrderService;

  @MessagePattern('create') // handles 'order.create'
  async create(data: CreateOrderDto, context: MessageContext) {
    return await this.orderService.create(data); // reply
  }

  @EventPattern('created') // handles 'order.created'
  async onCreated(event: OrderEvent, context: MessageContext) { /* ... */ }

  // Another service's vocabulary - opt out of the prefix
  @EventPattern({ pattern: 'payment.completed', prefix: false })
  async onPaymentCompleted(event: PaymentEvent, context: MessageContext) { /* ... */ }
}
```

## The Prefix Rule

The controller prefix is dot-joined onto **every** `@MessagePattern` **and** `@EventPattern` (`order` + `create` → `order.create`). A handler opts out with `prefix: false` and registers verbatim. The client side uses the same rule: `ulak.messages('order').emit('created')` publishes `order.created` — sender and listener always agree. Use `prefix: false` when the name is not yours: another service's events, a Kafka external topic, or a global catch-all.

**Warning (migrating from 0.7):** in 0.7 the prefix applied to `@MessagePattern` only; `@EventPattern` was always absolute. On a prefixed controller every `@EventPattern` now changes its subscription name — rewrite it relative to the prefix or add `prefix: false`. `@EventPattern('*')` under a prefix becomes `order.*`, no longer a global catch-all. A wildcard controller prefix (`@MessageController('order.*')`) is now a boot error. Asena prints the resolved patterns per controller at boot:

```
[Microservice] OrderHandler → "default" (prefix "order") msg: order.create | evt: order.created, payment.completed
```

## MessageContext

Second handler argument:

| Member | Description |
|---|---|
| `context.messageId` | Stable across redeliveries and identical for all handlers/groups of one emit — the dedup key. Kafka external topics: the `mid` header if present, else `topic:partition:offset` |
| `context.attempt` | Delivery attempt; `> 1` means retry. Kafka: broker-tracked in offset-commit metadata, survives crashes and rebalances |
| `context.headers` | Message headers. Kafka external topics: **all** raw record headers (e.g. a foreign `traceparent`) |

## Client API: ulak.messages

```typescript
import { Service } from '@asenajs/asena/decorators';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { ulak, type Ulak } from '@asenajs/asena/messaging';

@Service()
export class CheckoutService {
  @Inject(ulak.messages('order'))
  private orders: Ulak.Messages<'order'>;

  async checkout(dto: CheckoutDto) {
    // RPC - awaits the remote reply. send<T> defaults to unknown; name the reply type
    const order = await this.orders.send<{ id: string }>('create', dto); // → 'order.create'
    // Fire-and-forget event
    await this.orders.emit('created', { id: order.id });                 // → 'order.created'
  }
}
```

Options: `send(pattern, data, { timeout: 5000 })`; `emit(pattern, data, { headers: { 'x-tenant': 't-42' } })`. For absolute patterns inject the full `Ulak`:

```typescript
@Inject(ICoreServiceNames.__ULAK__)
private ulak: Ulak;

await this.ulak.send('inventory.reserve', { sku: 'A1' });
await this.ulak.emit('audit.recorded', entry);
```

## Transport Configuration

Return `{ microservice: <transport> }` from the `@Config` class's `transport()`. `serviceName` is **required** — it is the consumer group identity: all replicas of one service must share it; different services must differ (each service group gets its own copy of every event).

```typescript
import { Config } from '@asenajs/asena/decorators';
import { ConfigService } from '@asenajs/ergenecore'; // hono-adapter: same shape
import { RedisMicroserviceTransport } from '@asenajs/asena-redis';

@Config()
export class AppConfig extends ConfigService {
  transport() {
    return {
      microservice: new RedisMicroserviceTransport(
        { url: 'redis://localhost:6379' },
        { serviceName: 'order-service' },
      ),
    };
  }
}
```

Kafka — same shape, only the transport line changes (`@asenajs/asena-kafka`, peer `kafkajs` ^2.2, core 0.10.0+; broker pinned to Kafka 2.8–3.9):

```typescript
import { KafkaMicroserviceTransport } from '@asenajs/asena-kafka';

microservice: new KafkaMicroserviceTransport(
  { brokers: ['localhost:9092'] },
  { serviceName: 'order-service' },
),
```

**Note:** Both transports accept an existing service (`AsenaRedisService` / `AsenaKafkaService`) in place of the connection config — inject it into the config class and pass it as the first argument; connection settings then live in one place. Config classes are prepared after user components initialize, so the injected service is already connected when `transport()` runs. (Kafka: the transport borrows the adapter but creates its own producer/consumers/admin.)

## Redis Streams Transport

`RedisMicroserviceTransport` options:

| Option | Default | Description |
|---|---|---|
| `serviceName` | — (required) | Consumer group identity shared by replicas |
| `streamPrefix` | `'asena:ms'` | Key prefix for streams/channels |
| `requestTimeout` | `30000` | Default `send()` reply timeout (ms) |
| `maxRetries` | `3` | Event delivery attempts before DLQ (events only) |
| `claimIdleMs` | `60000` | Idle time before the sweep reclaims a pending entry — **handler duration must stay below this** |
| `maxStreamLength` | `100000` | `XADD MAXLEN ~` trim — bounds memory AND offline tolerance |
| `blockMs` | `5000` | `XREADGROUP BLOCK` duration |
| `count` | `16` | Max entries fetched per read |
| `maxInFlight` | `32` | Concurrent handler limit (backpressure) |
| `handlerTimeout` | `min(30000, claimIdleMs)` | Per-handler timeout; explicit values above `claimIdleMs` throw at construction. Timeout rejects the dispatch but the handler keeps running |
| `drainTimeout` | `10000` | Graceful drain window on shutdown |
| `commandTimeout` | `10000` | Watchdog for publisher commands; an overrunning command means a dead connection, which is replaced |

Mechanics — Redis keys:

| Purpose | Key | Mechanism |
|---|---|---|
| Events | `asena:ms:evt` (stream) | `XADD` + one consumer group per service; wildcards matched locally; at-least-once with ACK |
| Requests | `asena:ms:req:<pattern>` (stream per pattern) | Group distributes each request to exactly one replica |
| Replies | `asena:ms:reply:<instanceId>` (pub/sub) | Not durable |
| Dead letters | `asena:ms:dlq` (stream) | Poison events after `maxRetries` |

Redis-specific behavior:

- A background sweep (`XPENDING` + `XCLAIM`) rescues entries from crashed replicas and counts attempts. **Retry latency is sweep-driven, not immediate**: first retry ~60–90s with defaults; DLQ after `maxRetries` sweep cycles (minutes). Lower `claimIdleMs` if coming from BullMQ-style immediate retries.
- **Replies are not durable** (plain `PUBLISH`): a reply published while the caller has no live reply subscription is dropped — the caller gets `TIMEOUT`. Treat RPC spanning a Redis outage as retryable; make handlers idempotent, or use `emit()` where loss is unacceptable.
- `isConnected` (feeds the health endpoint) requires the publisher connection **and** an acknowledged reply subscription — an instance stays `503` slightly past its socket reconnect. Wire it into **readiness**, not liveness.
- If the consumer group is lost (Redis restart without persistence, `FLUSHALL`) it is re-created from the beginning of the stream; trimmed data is gone.
- No ordering guarantee — entries in a read batch dispatch in parallel.

## Kafka Client (produce/consume without microservices)

`@asenajs/asena-kafka` is also a plain Kafka client — if you only publish/read your own topics, this is all you need (skip the transport sections entirely). `@Kafka` registers the class as an injectable `@Service`; the connection opens on `server.start()` and closes on `server.stop()`:

```typescript
import { Kafka, AsenaKafkaService } from '@asenajs/asena-kafka';

@Kafka({
  config: { brokers: ['localhost:9092'], clientId: 'my-app' },
  name: 'AppKafka', // the injection key
})
export class AppKafka extends AsenaKafkaService {}
```

Produce with the inherited `sendMessage(topic, messages)` — values must be strings or Buffers, serialize yourself; same `key` → same partition → per-key ordering. Prefer adding domain methods (`publishAudit()`) on the `@Kafka` class over calling `sendMessage` from every caller.

Consume with `createConsumer({ groupId })` — the returned consumer is **caller-owned**: connect, subscribe, `run({ eachMessage })`, and disconnect it yourself, naturally from a `@Service`'s `@OnStart`/`@OnStop` (reverse stop order tears it down while `AppKafka` is still up). `groupId` is required; replicas sharing it split the partitions. `AsenaKafkaService`'s own `@OnStop` closes only the client and default producer — whatever `createProducer()`/`createConsumer()`/`createAdmin()` hand out stays yours to close. Full client API: `https://asena.sh/raw/packages/kafka.md`.

## Kafka Transport

`KafkaMicroserviceTransport` options:

| Option | Default | Description |
|---|---|---|
| `serviceName` | — (required) | Consumer group identity (actual group id is prefix-scoped: `asena.ms.order-service`) |
| `topicPrefix` | `'asena.ms'` | Topic/group namespace (`[a-zA-Z0-9._-]` only) |
| `requestTimeout` | `30000` | Default `send()` reply timeout (ms) |
| `maxRetries` | `3` | Event delivery attempts before DLQ (events only) |
| `retryBackoffMs` | `5000` | Pause before a failed event's partition is resumed for the retry fetch |
| `handlerTimeout` | `min(30000, sessionTimeout)` | Values above `sessionTimeout` throw at construction (kafkajs cannot heartbeat during a handler) |
| `maxInFlight` | `16` | Partitions processed concurrently (per-partition stays sequential) |
| `drainTimeout` | `10000` | Graceful drain window on shutdown |
| `sessionTimeout` | `30000` | Group session timeout — crash detection speed |
| `heartbeatInterval` | `3000` | Keep ≤ 1/3 of `sessionTimeout` |
| `rebalanceTimeout` | `60000` | Max rebalance duration |
| `maxWaitTimeInMs` | `1000` | Idle fetch long-poll |
| `eventPartitions` / `requestPartitions` / `replyPartitions` | `4` | Partition counts for transport-created topics |
| `replicationFactor` | `-1` | Broker default |
| `healthCheckIntervalMs` | `5000` | Active broker probe interval (one of two inputs to `isConnected`) |
| `external` | — | Foreign-topic interop: `{ topics: (string \| { name, keyHeader? })[], fromBeginning? }` |

Topics it owns (created explicitly, leaders awaited): events `asena.ms.evt` (shared), requests `asena.ms.req.<pattern>` (per pattern), replies `asena.ms.reply` (shared, filtered by correlationId), DLQ `asena.ms.dlq`.

Kafka-specific behavior:

- `context.attempt` is persisted in offset-commit metadata (two commits per record) — trustworthy across crashes and rebalances, no side store.
- Failed event: offset stays uncommitted, partition paused + seeked back, re-fetched after `retryBackoffMs` — the broker genuinely redelivers.
- **First boot starts at latest**: a brand-new group ignores messages produced before the service existed — deploy consumers before producers. Position is pinned to a committed offset immediately, so crash-restarts never skip records.
- Requests older than the caller's own timeout are dropped without execution (no dead-RPC backlog after restart).
- `isConnected` requires a passing metadata probe **and** a reply consumer that has rejoined and is fetching — after an outage the instance can stay `503` for tens of seconds while the reply group rejoins; a `send()` before that would time out. Liveness probes must be more forgiving than readiness. Replies themselves survive a rejoin (delivered late, never skipped).
- Ordering: not preserved end-to-end with the default 4 event partitions; use `eventPartitions: 1` for strict ordering. Handlers must finish below `sessionTimeout` or the member is evicted and the record concurrently redelivered.

## Kafka External Topics (Interop)

The `external` option consumes/produces **envelope-less foreign topics** (Quarkus, CDC, plain producers) through the same decorators:

```typescript
new KafkaMicroserviceTransport({ brokers }, {
  serviceName: 'billing-service',
  external: {
    topics: ['orders', { name: 'invoices', keyHeader: 'x-tenant' }],
    fromBeginning: false, // start position on FIRST subscribe (default: latest)
  },
});
```

```typescript
@MessageController()
export class OrdersListener {
  @EventPattern({ pattern: 'orders', prefix: false }) // pattern = the topic name, verbatim
  async onOrder(order: any, context: MessageContext) { /* ... */ }
}

await this.ulak.emit('invoices', { invoiceId: 'inv-1' }, { headers: { 'x-tenant': 't-42' } });
```

Rules: **always `prefix: false`** on a prefixed controller — otherwise the handler registers as `billing.orders` and the foreign topic is silently never consumed (service boots green; `listen()` prints a hint). Outbound values are plain JSON, headers verbatim, `keyHeader`'s value becomes the record key. External topics are **event-only** — `send()` to one rejects with `SEND_FAILED`. The transport never creates external topics; missing subscribed topics fail boot loudly. Retry/DLQ/`context.attempt` work as for own topics; position-derived messageIds change on replay-tool re-produces, so carry your own dedup key (or `mid` header) for replayed records.

## Delivery Semantics

| Aspect | Events (`@EventPattern`) | RPC (`@MessagePattern`) |
|---|---|---|
| Guarantee | **At-least-once** (consumer groups + ack/commit) | At-most-once per attempt |
| Handler error | No ack → redelivered up to `maxRetries`, then **DLQ** | **Final**: `ok:false` reply + ack — no broker retry |
| Service offline | Messages wait in the stream/topic | Caller times out (`UlakErrorCode.TIMEOUT`) |
| Crash mid-handling | Another replica takes over (Redis: XCLAIM; Kafka: rebalance) | Rescued only if the caller hasn't timed out |

Operational rules:

- Handler duration below `claimIdleMs` (Redis) / `sessionTimeout` (Kafka), or a second replica processes the same entry concurrently.
- `maxStreamLength` bounds offline tolerance — events older than the trim window are lost to services down too long.
- **`@OnStart` cannot send messages** — `ulak.send`/`emit` there throws `NO_TRANSPORT` (transports are wired during application setup, after start hooks). Defer the first publish. `@OnStop` **can** publish (transports torn down after stop hooks).

## Idempotency and DLQ

At-least-once means duplicates are normal (crash after handling, before ACK). Event handlers must be idempotent — dedup on `context.messageId`:

```typescript
@EventPattern('payment.completed')
async onPayment(event: PaymentEvent, context: MessageContext) {
  if (await this.processedStore.has(context.messageId)) return; // duplicate
  await this.orderService.markPaid(event.orderId);
  await this.processedStore.add(context.messageId);
}
```

Monitor the DLQ (Redis stream `asena:ms:dlq` / Kafka topic `asena.ms.dlq`): poison events land there after `maxRetries` with provenance (`origin_stream`, `origin_group`, `delivery_count` on Redis; provenance headers with original value preserved on Kafka).

## Multiple Named Transports

```typescript
transport() {
  return {
    microservice: {
      default: new RedisMicroserviceTransport({ url }, { serviceName: 'order-service' }),
      analytics: new KafkaMicroserviceTransport({ brokers }, { serviceName: 'order-service' }),
    },
  };
}
```

```typescript
@MessageController({ prefix: 'metrics', transport: 'analytics' })   // handler side
export class AnalyticsHandler { /* ... */ }

@Inject(ulak.messages('metrics', { transport: 'analytics' }))       // client side
private metrics: Ulak.Messages<'metrics'>;
```

A single unnamed transport registers as `default`. Binding a controller to an unknown transport name fails at boot listing the configured names. A service with zero handlers on a transport boots it client-only (no consumer group). The health endpoint reports each transport separately (`transports: { default: 'connected', analytics: 'connected' }`) — a partial broker outage degrades to 503 while the other keeps flowing.

## Headless Mode and Health

A service can run without an HTTP server — driven purely by messages, events and schedules:

```typescript
import { AsenaServerFactory } from '@asenajs/asena';

const server = await AsenaServerFactory.create({
  headless: true,          // explicit opt-in - omitting the adapter alone throws
  logger,
  health: { port: 9090 },  // optional K8s probe endpoint
});
await server.start(); // no HTTP port opened
```

Runs: configs, microservice layer, in-process events, scheduled tasks. Ignored with a warning: `@Controller`, `@WebSocket`, `@FrontendController`. In headless mode `keepAlive` defaults to `true` (the process stays alive without a listening socket); pass `keepAlive: false` to opt out.

Health routes (`path` defaults to `/healthz`, minimal `Bun.serve`):

| Route | Behavior |
|---|---|
| `GET /healthz/live` | `200` while the process is alive; touches no dependency → point the restart policy here |
| `GET /healthz/ready` | `200` while serving and all transports connected; `503 degraded` if any transport disconnected; `503 not_ready` while starting/stopping → point the load balancer here |
| `GET /healthz` | Same body as `/ready` |

Readiness flips to `503` the moment `stop()` begins; the health server is taken down last, so the instance is out of rotation for the whole drain.

## Graceful Shutdown

Transports are taken down **after** `@OnStop` hooks (a hook can still publish) and before `ulak.dispose()`. Per-transport drain: (1) consuming stops, (2) in-flight handlers get `drainTimeout` (default 10s) — completed ones ACK, unfinished stay pending for another replica, (3) pending `send()` calls are rejected, (4) the consumer's group registration is removed only if it has no pending entries, (5) connections close.

```typescript
await server.stop({ drainTimeout: 30_000 }); // 0.10.0+; the old boolean form still works
```

**Note:** `handlerTimeout` is not cancellation — the dispatch is rejected (entry stays un-ACKed for redelivery) but the handler function keeps running until it settles.

## InMemoryTransport (Testing)

Zero-dependency loopback in core — for development before a broker exists and for integration tests:

```typescript
import { InMemoryTransport } from '@asenajs/asena/microservice';

transport() {
  return { microservice: new InMemoryTransport() };
}
```

**Warning:** in-memory `emit()` awaits its handlers — a test relying on "handled immediately after emit" passes here and races against Redis/Kafka in production. Await an observable effect instead. All handlers of one emit see the same `messageId`, matching broker semantics. `send()` with no registered handler fails with `SEND_FAILED`.

## UlakError Codes

```typescript
import { UlakError, UlakErrorCode } from '@asenajs/asena/messaging';

try {
  const res = await this.orders.send('create', dto, { timeout: 5000 });
} catch (error) {
  if (error instanceof UlakError && error.code === UlakErrorCode.TIMEOUT) { /* retry (idempotent!) or fail */ }
}
```

| Code | Meaning |
|---|---|
| `NO_TRANSPORT` | `send`/`emit` called but no microservice transport configured (includes calls inside `@OnStart`) |
| `TRANSPORT_NOT_FOUND` | Named transport does not exist (message lists available names) |
| `TIMEOUT` | No reply within the caller's timeout (`send`) |
| `REMOTE_ERROR` | Remote handler threw **or exceeded `handlerTimeout`** — responder replies `ok:false`, so the caller sees `REMOTE_ERROR`, not `TIMEOUT` |
| `SEND_FAILED` | Broker publish failed / no handler registered (InMemoryTransport) / `send()` to a Kafka external topic |

## Custom Transport SPI

Implement `MicroserviceTransport` to integrate any broker:

```typescript
import type { MicroserviceTransport } from '@asenajs/asena/microservice';

export class MyBrokerTransport implements MicroserviceTransport {
  readonly name = 'my-broker';
  get isConnected(): boolean { /* ... */ }
  async init() {}                                  // connect; send/emit usable after this
  registerMessageHandler(pattern, handler) {}      // exact patterns (RPC)
  registerEventHandler(pattern, handler) {}        // absolute patterns, wildcards allowed
  async listen() {}                                // consume ONLY sources with handlers
  async send(pattern, data?, options?) {}          // request/response with correlation
  async emit(pattern, data?, options?) {}          // fire-and-forget
  async destroy(options?) {}                       // graceful drain, then release
}
```

Contract: `init()` must make `send`/`emit` usable **before** `listen()` (client-only mode); with zero registered handlers `listen()` must not start any consumer; wildcard matching can reuse `matchesEventPattern` from `@asenajs/asena/event`. Full detail: `https://asena.sh/raw/concepts/microservices.md`.
