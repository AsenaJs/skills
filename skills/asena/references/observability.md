# Observability

Logging with `@asenajs/asena-logger` and tracing/metrics with `@asenajs/asena-otel`.

## Contents
- AsenaLogger
- OpenTelemetry Quickstart
- Auto-Tracing
- OtelService API
- HTTP Middleware Spans & Metrics
- Sampling & Route Exclusion
- Outgoing Calls & Messaging Interceptor
- Shutdown & Testing

## AsenaLogger

```bash
bun add @asenajs/asena-logger
```

Winston-based; implements the `ServerLogger` shape. As published (2.0.0) it peers `@asenajs/asena` `^0.10.0`. Requires Bun >= 1.3.12.

**Standard pattern** — export one instance globally and import it everywhere:

```typescript
// src/logger.ts
import { AsenaLogger } from '@asenajs/asena-logger';
export const logger = new AsenaLogger();
```

Constructor: `new AsenaLogger(winstonLogger?, options?)`. The options object is the simplest way to change level or add transports:

```typescript
export const logger = new AsenaLogger(undefined, {
  level: 'debug',
  transports: [/* extra Winston transports */],
});
```

Full Winston control (file transports, formats, rotation — a Loki transport goes here the same way):

```typescript
import winston from 'winston';
const winstonLogger = winston.createLogger({
  level: 'info',
  transports: [
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' }),
    new winston.transports.Console(),
  ],
});
export const logger = new AsenaLogger(winstonLogger);
```

Alternative to the global export — IoC injection (better for mocking in tests):

```typescript
import { ICoreServiceNames } from '@asenajs/asena/ioc/types';
import type { ServerLogger } from '@asenajs/asena/logger';

@Inject(ICoreServiceNames.SERVER_LOGGER)
private logger!: ServerLogger;
```

### API

| Method | Use |
|---|---|
| `info(message, meta?)` | `logger.info('User logged in', { userId: 123 })` |
| `error(message, meta?)` | `meta` may be an `Error` — stack is printed |
| `warn(message, meta?)` | |
| `debug(message, meta?)` | |
| `log(level, message, meta?)` | Custom level, e.g. `logger.log('verbose', '...', { step: 1 })` |
| `profile(id)` | Call once to start, again with the same id to stop and log elapsed ms |

Levels are Winston's `npm` set, priority 0–6: `error`, `warn`, `info`, `http`, `verbose`, `debug`, `silly`. Only `error`/`warn`/`info`/`debug` are colored; others print uppercased, uncolored.

## OpenTelemetry Quickstart

```bash
bun add @asenajs/asena-otel @opentelemetry/api @opentelemetry/resources @opentelemetry/sdk-trace-base @opentelemetry/sdk-metrics @opentelemetry/semantic-conventions @opentelemetry/context-async-hooks
# OTLP exporters (Jaeger, Grafana Tempo, ...):
bun add @opentelemetry/exporter-trace-otlp-http @opentelemetry/exporter-metrics-otlp-http
```

Requires Bun >= 1.3.12, `@asenajs/asena` >= 0.10.0. One runtime dependency (`@opentelemetry/context-async-hooks`); the rest are peers.

Three steps — all three are required for HTTP tracing:

```typescript
// 1) src/otel/AppOtel.ts — auto-discovered by the IoC container
import { Otel, OtelTracingPostProcessor } from '@asenajs/asena-otel';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';

@Otel({
  serviceName: 'my-app',
  serviceVersion: '1.0.0',
  traceExporter: new OTLPTraceExporter({ url: 'http://localhost:4318/v1/traces' }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({ url: 'http://localhost:4318/v1/metrics' }),
  }),
  autoTrace: { services: true, controllers: true },
})
export class AppOtel extends OtelTracingPostProcessor {}
```

```typescript
// 2) src/middlewares/AppOtelMiddleware.ts — REQUIRED local wrapper:
// the container only scans src, so OtelTracingMiddleware in node_modules is
// undiscoverable and the container errors without this class
import { Middleware } from '@asenajs/asena/decorators';
import { OtelTracingMiddleware } from '@asenajs/asena-otel';

@Middleware()
export class AppOtelMiddleware extends OtelTracingMiddleware {}
```

```typescript
// 3) register it in the @Config class
public globalMiddlewares() {
  return [AppOtelMiddleware /* , other middleware */];
}
```

### AsenaOtelOptions

| Field | Type | Default | Description |
|---|---|---|---|
| `serviceName` | `string` | — | Required |
| `serviceVersion` | `string` | `'0.0.0'` | Resource attribute |
| `traceExporter` | `SpanExporter` | — | Required (OTLP, Jaeger, in-memory, ...) |
| `metricReader` | `MetricReader` | — | Enables metrics collection |
| `autoTrace` | `{ services?: boolean; controllers?: boolean }` | `{}` (both `false`) | Auto-wrap component methods |
| `sampler` | `Sampler` | — | Custom sampling |
| `ignoreRoutes` | `string[]` | `[]` | Paths excluded from tracing |

## Auto-Tracing

With `autoTrace` enabled, `OtelTracingPostProcessor` wraps component methods in a Proxy; each call creates an `INTERNAL` span named `"{ClassName}.{methodName}"`, parented under the middleware's `SERVER` span via `context.with()` — one request produces a full waterfall (`GET /api/users` → `UserController.list` → `UserService.getAll`) sharing one `traceId`.

| Decorator | Auto-traced |
|---|---|
| `@Service`, `@Controller` | Yes |
| `@Repository`, `@Redis` | Yes (treated as Service) |
| `@Middleware`, `@Component` | No |

Methods starting with `_`, constructors, and Symbol-keyed methods are always skipped.

## OtelService API

`OtelService` is an injectable `@Service`, auto-discovered — `@Inject('OtelService')`.

| Member | Signature | Description |
|---|---|---|
| `withSpan(name, fn)` | `(name: string, fn: (span: Span) => Promise<T>) => Promise<T>` | Create span, auto-end, record errors/status |
| `getActiveSpan()` | `() => Span \| undefined` | Current active span |
| `injectTraceContext(headers?)` | `(headers?: Record<string, string>) => Record<string, string>` | Add W3C `traceparent` (`00-{traceId}-{spanId}-{traceFlags}`) to a headers object |
| `tracer` | `Tracer` | Raw OpenTelemetry tracer |
| `meter` | `Meter` | Raw OpenTelemetry meter |

```typescript
async processOrder(orderId: string) {
  return this.otelService.withSpan('process-order', async (span) => {
    span.setAttribute('order.id', orderId);
    return { success: true };
  });
}
```

Direct tracer/meter fields via expression injection: `@Inject('OtelService', (s) => s.tracer) private tracer: Tracer;`

## HTTP Middleware Spans & Metrics

Per request, `OtelTracingMiddleware` creates a `SERVER` span named `"{METHOD} {PATH}"`, renamed to the route pattern after matching (`GET /api/users/:id`) for low cardinality. It also extracts incoming W3C `traceparent` headers (distributed tracing in).

Span attributes: `http.request.method`, `url.path`, `http.route` (pattern, after matching), `http.response.status_code`.

Metrics (needs `metricReader`): `http.server.request.count` (counter), `http.server.request.duration` (histogram, ms) — both by method/path.

## Sampling & Route Exclusion

```typescript
import { ratioBasedSampler } from '@asenajs/asena-otel';

@Otel({
  serviceName: 'my-app',
  traceExporter: exporter,
  sampler: ratioBasedSampler(0.1),                    // sample 10% of root traces
  ignoreRoutes: ['/health', '/metrics', '/admin/*'],  // no spans, no metrics
})
export class AppOtel extends OtelTracingPostProcessor {}
```

- `ratioBasedSampler(ratio)` = `ParentBasedSampler` wrapping `TraceIdRatioBasedSampler`; children respect the parent's decision. Always set a sampler in production; `0.1` is a sane start.
- `ignoreRoutes`: exact match (`/health`) or wildcard suffix (`/admin/*` covers `/admin/` and sub-paths). **Warning:** matched against the URL path **before** route matching — use static paths, not route patterns.

## Outgoing Calls & Messaging Interceptor

**Outgoing `fetch` is NOT auto-instrumented.** Propagate context manually:

```typescript
const headers = this.otelService.injectTraceContext({ 'Content-Type': 'application/json' });
const res = await fetch('http://payment-service/api/charge', {
  method: 'POST', headers, body: JSON.stringify(payload),
});
// wrap in withSpan('call-payment-service', ...) for a dedicated client span
```

For the microservice layer, propagation is automatic via the `otelMessaging()` interceptor:

```typescript
import { otelMessaging } from '@asenajs/asena-otel';

public transport() {
  return {
    microservice: new RedisMicroserviceTransport(
      { url: 'redis://localhost:6379' },
      { serviceName: 'order-service' },
    ),
    interceptors: [otelMessaging({ system: 'redis' })],
  };
}
```

- Outgoing `send`/`emit`: `PRODUCER` span (`send order.create` / `publish order.created`), injects `traceparent`/`tracestate` into message headers; RPC spans carry full round-trip duration and error state.
- Incoming handlers: run inside a `CONSUMER` span (`process order.create`) with upstream context extracted — one distributed trace across services.
- Attributes: `messaging.system`, `messaging.destination.name`, `messaging.operation.type`, `messaging.message.id`; redeliveries set `messaging.redelivery_count`.
- Register the interceptor in **every** service (producers and consumers) to keep the chain unbroken.

## Shutdown & Testing

`server.stop()` drives `shutdown()` on the `@Otel` class through an inherited `@OnStop`: buffered spans/metrics flush, exporter timers stop, context manager unhooks. Nothing to call or register; failures are logged, not thrown; `shutdown()` is idempotent. **Warning (0.10.0+):** the package no longer installs its own `SIGTERM`/`SIGINT` handlers — signals are handled by the server. Remove any custom `SIGTERM` flush workaround.

For tests, use `InMemorySpanExporter` / `InMemoryMetricExporter(AggregationTemporality.CUMULATIVE)` as exporters, then flush before asserting (`BatchSpanProcessor` is lazy): `await (trace.getTracerProvider() as any).forceFlush?.()` then `spanExporter.getFinishedSpans()`; for metrics `await metricReader.forceFlush()`. In OTel SDK v2 assert parent links via `span.parentSpanContext?.spanId` (not `parentSpanId`).

Deeper detail: `https://asena.sh/raw/packages/opentelemetry.md`.
