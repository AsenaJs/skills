# AsenaJS Docs Map

Index of the full official documentation. Fetch any page as raw markdown from its URL when the bundled references do not cover an API in enough depth. Machine index: `https://asena.sh/llms.txt`.

## Docs

| Page | URL | Covers |
|---|---|---|
| Get Started | https://asena.sh/raw/get-started.md | Install, TypeScript decorator config, first controller and service |
| Examples | https://asena.sh/raw/examples.md | Complete worked examples: REST APIs, WebSocket chat, auth, database |
| Philosophy | https://asena.sh/raw/philosophy.md | Design principles and deliberate trade-offs |
| Roadmap | https://asena.sh/raw/roadmap.md | Completed, active, and planned features |
| Showcase | https://asena.sh/raw/showcase.md | Production applications built with Asena |

## Concepts

| Page | URL | Covers |
|---|---|---|
| Controllers | https://asena.sh/raw/concepts/controllers.md | Decorator-based routing and DI in HTTP controllers |
| Services | https://asena.sh/raw/concepts/services.md | @Service, singleton/prototype scopes, lifecycle hooks |
| Dependency Injection | https://asena.sh/raw/concepts/dependency-injection.md | IoC container, field-based @Inject, @Strategy |
| Component Lifecycle | https://asena.sh/raw/concepts/lifecycle.md | @OnStart/@OnStop, shutdown ordering, signals, health probes |
| Middleware | https://asena.sh/raw/concepts/middleware.md | Global, pattern-based, controller- and route-level middleware |
| Context API | https://asena.sh/raw/concepts/context.md | Unified request/response handling across adapters |
| Validation | https://asena.sh/raw/concepts/validation.md | Type-safe request validation with Zod |
| Static Files | https://asena.sh/raw/concepts/static-files.md | @StaticServe, lifecycle hooks, path configuration |
| WebSocket | https://asena.sh/raw/concepts/websocket.md | Real-time namespaces, rooms, typed socket data |
| Event System | https://asena.sh/raw/concepts/event-system.md | In-process fire-and-forget events, wildcard matching |
| Microservices | https://asena.sh/raw/concepts/microservices.md | Broker-agnostic messaging, request/response, headless mode, delivery semantics |
| Ulak | https://asena.sh/raw/concepts/ulak.md | Centralized WebSocket message broker |
| Scheduled Tasks | https://asena.sh/raw/concepts/scheduled-tasks.md | Cron jobs with @Schedule and CronRunner |
| Frontend Controller | https://asena.sh/raw/concepts/frontend-controller.md | HTML pages with @FrontendController/@Page and Bun HTML imports |
| PostProcessor | https://asena.sh/raw/concepts/post-processor.md | Intercept and transform components during IoC bootstrap |
| Inheritance | https://asena.sh/raw/concepts/inheritance.md | Decorator behavior across class hierarchies, override resolution |

## Adapters

| Page | URL | Covers |
|---|---|---|
| Adapters Overview | https://asena.sh/raw/adapters/overview.md | Adapter system, choosing an adapter |
| Ergenecore | https://asena.sh/raw/adapters/ergenecore.md | Native Bun adapter, zero dependencies, built-in middleware |
| Hono Adapter | https://asena.sh/raw/adapters/hono.md | Hono-based adapter, ecosystem compatibility |

## Official Packages

| Page | URL | Covers |
|---|---|---|
| AsenaLogger | https://asena.sh/raw/packages/logger.md | Winston-based logging, transports (console, file, Loki), profiling |
| Asena Drizzle | https://asena.sh/raw/packages/drizzle.md | Type-safe database integration, Repository pattern |
| Asena OpenAPI | https://asena.sh/raw/packages/openapi.md | OpenAPI 3.1 generation from validators, Swagger UI |
| Asena OpenTelemetry | https://asena.sh/raw/packages/opentelemetry.md | HTTP tracing, auto-tracing, metrics, distributed tracing |
| Asena Redis | https://asena.sh/raw/packages/redis.md | @Redis client, caching decorators, multi-pod WS transport, Streams microservice transport |
| Asena Kafka | https://asena.sh/raw/packages/kafka.md | Produce/consume client and Kafka microservice transport |

## CLI

| Page | URL | Covers |
|---|---|---|
| CLI Overview | https://asena.sh/raw/cli/overview.md | Scaffolding, code generation, dev server, bundling |
| CLI Installation | https://asena.sh/raw/cli/installation.md | Global install with Bun, verification |
| CLI Commands | https://asena.sh/raw/cli/commands.md | create, generate, dev start, build, init, shortcuts |
| CLI Configuration | https://asena.sh/raw/cli/configuration.md | asena-config.ts full reference |
| Suffix Configuration | https://asena.sh/raw/cli/suffix-configuration.md | Component naming-convention customization |
| CLI Examples | https://asena.sh/raw/cli/examples.md | Step-by-step project tutorial |

## Guides

| Page | URL | Covers |
|---|---|---|
| Configuration | https://asena.sh/raw/guides/configuration.md | @Config: global middleware, error handling, env vars, transports |
| Error Handling | https://asena.sh/raw/guides/error-handling.md | HttpException, global handlers, custom errors, duplicate-module forensics |
| Deployment | https://asena.sh/raw/guides/deployment.md | Production build and deployment |

## Testing

| Page | URL | Covers |
|---|---|---|
| Testing Overview | https://asena.sh/raw/testing/overview.md | Built-in testing utilities, Bun test runner |
| MockComponent API | https://asena.sh/raw/testing/mock-component.md | Automatic dependency mocking reference |
| createTestApp | https://asena.sh/raw/testing/test-app.md | Boot the full app in a test, override components, assert HTTP |
| createWebTest | https://asena.sh/raw/testing/web-test.md | Web layer only: real controllers/middleware, auto-mocked rest |
| Testing Examples | https://asena.sh/raw/testing/examples.md | Patterns for controllers, services, WebSockets, middleware |
