# WebSocket & Ulak

WebSocket namespaces (`@WebSocket` + `AsenaWebSocketService`), room pub/sub, the Ulak message broker, and the Redis multi-pod WebSocket transport.

## Contents

- WebSocket Namespace
- Lifecycle Methods
- Socket API
- Rooms and Broadcasting
- WebSocket Middleware
- Pitfalls
- Ulak: WebSocket Message Broker
- Ulak Error Handling
- Multi-Pod: RedisTransport

## WebSocket Namespace

Extend `AsenaWebSocketService<T>` (`T` types `ws.data.values`) and decorate with `@WebSocket`:

```typescript
import { WebSocket } from '@asenajs/asena/decorators';
import { AsenaWebSocketService, type Socket } from '@asenajs/asena/web-socket';

interface UserData { userId: string; username: string }

@WebSocket({ path: '/chat', name: 'ChatSocket' })
export class ChatSocket extends AsenaWebSocketService<UserData> {
  protected async onOpen(ws: Socket<UserData>): Promise<void> {
    ws.subscribe(`user:${ws.data.values.userId}`);
    ws.send('Welcome!');
  }

  protected async onMessage(ws: Socket<UserData>, message: string): Promise<void> {
    ws.send(`Echo: ${message}`);
  }

  protected async onClose(ws: Socket<UserData>): Promise<void> {}
}
```

`@WebSocket` options: `path` (route), `name` (IoC component name), `middlewares` (middleware class array). String shorthand `@WebSocket('/chat')` also works.

Socket data shape — Asena fills `id` and `path`; `values: T` is user-managed (set via `context.setWebSocketValue()` in middleware, synced to `socket.data.values`):

```typescript
export interface WebSocketData<T = any> {
  values: T;
  id: string;
  path: string;
}
```

WebSocket services are normal IoC components: inject them into services/controllers (`@Inject(ChatSocket)`) and call `this.chatSocket.to(room, data)` / `.in(data)` from anywhere — unless that creates a circular dependency; then use Ulak (below).

## Lifecycle Methods

Declared `protected`, may be async. Only these are documented:

| Method | Called when |
|---|---|
| `onOpen(ws: Socket<T>)` | Client connects |
| `onMessage(ws: Socket<T>, message: string \| Buffer)` | Message received |
| `onClose(ws: Socket<T>)` | Client disconnects |

On close Asena removes the socket from `this.sockets` and Bun drops all its topic subscriptions — use `onClose` only for your own state.

## Socket API

| Member | Description |
|---|---|
| `ws.send(message)` | Send to this client. Raw passthrough — pass a string/serialized payload yourself |
| `ws.id` | Unique socket id |
| `ws.data` | `WebSocketData<T>` (see above) |
| `ws.subscribe(room)` / `ws.unsubscribe(room)` | Join/leave a room (Bun pub/sub topic, tracked automatically) |
| `ws.publish(room, data)` / `publishText` / `publishBinary` | Broadcast to room subscribers, **excluding the sender**. Raw passthrough |

Service-level members:

| Member | Description |
|---|---|
| `this.sockets` | Map of connected sockets in this namespace (`.get(id)`, `.keys()`) |
| `this.to(room, data)` | Broadcast to a room, sender included. **Serializes internally** — pass objects |
| `this.in(data)` | Broadcast to all connected clients of the namespace. Serializes internally |

## Rooms and Broadcasting

`subscribe`/`unsubscribe`/`publish` are the built-in room system — never keep a manual `Map<string, Socket[]>` for delivery:

```typescript
protected async onOpen(ws: Socket<ChatData>): Promise<void> {
  const room = ws.data.values.room || 'general';
  ws.subscribe(room);
  ws.publish(room, JSON.stringify({ type: 'user_joined' })); // others only
}
```

Private message: `this.sockets.get(socketId)?.send(payload)`.

**Warning:** Rooms are not enumerable. Membership lives in Bun's pub/sub topics — there is no `this.rooms` map and no `getSocketsByRoom()`. For a roster/count, keep your own registry (add socket id in `onOpen`, delete in `onClose`). That registry is per-process: in multi-pod deployments each pod sees only its own sockets; a global count needs a shared store (Redis).

## WebSocket Middleware

Same middleware contract as controllers; runs in array order **before** the upgrade. Reject by **returning** a response (e.g. `return context.send({...}, 401)`) or `false` (mapped to 403). Merely omitting `next()` does NOT reject on ergenecore — the upgrade still proceeds; always return something to block it.

```typescript
import { Middleware } from '@asenajs/asena/decorators';
import { MiddlewareService, type Context } from '@asenajs/ergenecore'; // hono-adapter exports the same

@Middleware()
export class WsAuthMiddleware extends MiddlewareService {
  async handle(context: Context, next: () => Promise<void>): Promise<boolean | Response> {
    const token = new URL(context.req.url).searchParams.get('token');
    if (!token) return context.send({ error: 'Unauthorized' }, 401);
    context.setWebSocketValue({ userId: '123', username: 'john' }); // → ws.data.values
    await next();
  }
}

@WebSocket({ path: '/private', middlewares: [WsAuthMiddleware], name: 'PrivateSocket' })
export class PrivateSocket extends AsenaWebSocketService<UserData> { /* ... */ }
```

## Pitfalls

- **`ws.publish()` excludes the sender** (also `publishText`/`publishBinary`). If the sender must see the message, `ws.send(payload)` it explicitly, or use `this.to()`/`this.in()`, which include the sender. This rule does not change when a `transport()` is configured (before `@asenajs/asena` 0.10.1 it did — a transport silently echoed publishes back to the sender).
- **Do not pre-stringify for `this.to()`/`this.in()`** — they serialize internally; a JSON string gets double-encoded and the client receives a quoted string. `ws.send()`/`ws.publish()` are the opposite: raw passthrough, pass strings.
- **No manual cleanup in `onClose`** — `this.sockets.delete(ws.id)` and `ws.unsubscribe(...)` are done for you (Bun drops every subscription on close).
- **Rooms are not enumerable** — see above; keep your own registry.

## Ulak: WebSocket Message Broker

Ulak is the in-process broker that lets services/controllers/jobs send WebSocket messages **without injecting the WebSocket service** — breaking the circular dependency `WebSocket ↔ Service`. (Ulak is also the microservice client: `ulak.send()`/`ulak.emit()`/`ulak.messages()` — see `https://asena.sh/raw/concepts/microservices.md`.)

### Injection Patterns

```typescript
import { Service } from '@asenajs/asena/decorators';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { ulak, type Ulak } from '@asenajs/asena/messaging';
import { ICoreServiceNames } from '@asenajs/asena/ioc/types';

@Service('UserService')
export class UserService {
  // 1. Scoped namespace (recommended)
  @Inject(ulak('/notifications'))
  private notifications: Ulak.NameSpace<'/notifications'>;

  // 2. Expression-based
  @Inject(ICoreServiceNames.__ULAK__, (u: Ulak) => u.namespace('/chat'))
  private chat: Ulak.NameSpace<'/chat'>;

  // 3. Direct Ulak (multiple/dynamic namespaces; must pass namespace per call)
  @Inject(ICoreServiceNames.__ULAK__)
  private ulak: Ulak;

  async createUser(name: string) {
    await this.notifications.broadcast({ type: 'user_created', name });
    await this.notifications.to(`user:123`, { type: 'notification' });
  }
}
```

The namespace path is the `@WebSocket` path (`ulak('/notifications')` ↔ `@WebSocket({ path: '/notifications' })`).

### API

Core `Ulak` (namespace passed per call):

```typescript
class Ulak {
  broadcast(namespace: string, data: any): Promise<void>;
  to(namespace: string, room: string, data: any): Promise<void>;
  toSocket(namespace: string, socketId: string, data: any): Promise<void>;
  toMany(namespace: string, rooms: string[], data: any): Promise<void>;   // parallel
  broadcastAll(data: any): Promise<void>;                                  // every namespace
  bulkSend(operations: BulkOperation[]): Promise<BulkResult>;
  namespace<T extends string>(path: T): Ulak.NameSpace<T>;
  getNamespaces(): string[];
  hasNamespace(namespace: string): boolean;
  getSocketCount(namespace: string): number;
  unregisterNamespace(path: string): void;
  dispose(): void;
}
```

Scoped `Ulak.NameSpace<T>`: same methods minus the `namespace` argument — `broadcast(data)`, `to(room, data)`, `toSocket(socketId, data)`, `toMany(rooms, data)`, `getSocketCount()`, plus readonly `path`.

`bulkSend` operations: `{ type: 'broadcast' | 'room' | 'socket', namespace, room?, socketId?, data }`; result: `{ total, succeeded, failed, results }`.

**Warning:** `toMany()`, `broadcastAll()` and `bulkSend()` never reject — they use `Promise.allSettled` and log failures (`bulkSend` resolves with counts). Only the single-target methods (`broadcast`, `to`, `toSocket`) throw `UlakError`.

**Note:** `ulak.dispose()` is called by `AsenaServer.stop()` as of `@asenajs/asena` 0.10.0 (up to 0.9.x tests/hot-reload had to call it manually). Calling it yourself is harmless.

### Testing

`mockComponent(ChatService)` replaces `ulak()` injections with deep mocks (`mocks.chat.to` etc. are assertable Bun mocks); `createTestUlakStub('/chat')` from `@asenajs/asena/test` gives a fully typed override.

## Ulak Error Handling

```typescript
import { UlakError, UlakErrorCode } from '@asenajs/asena/messaging';

try {
  await this.chat.to(roomId, { message });
} catch (error) {
  if (error instanceof UlakError && error.code === UlakErrorCode.NAMESPACE_NOT_FOUND) { /* ... */ }
}
```

`UlakError` fields: `code`, `namespace?`, `cause?`.

WebSocket codes: `NAMESPACE_NOT_FOUND`, `NAMESPACE_ALREADY_EXISTS`, `INVALID_NAMESPACE`, `INVALID_MESSAGE`, `SEND_FAILED`, `BROADCAST_FAILED`, `SOCKET_NOT_FOUND`, `SERVICE_NOT_INITIALIZED`.

Microservice codes (`send`/`emit`/`messages()`): `NO_TRANSPORT`, `TRANSPORT_NOT_FOUND`, `TIMEOUT`, `REMOTE_ERROR`.

**Warning:** `ulak.send()`/`emit()` inside an `@OnStart` hook throws `NO_TRANSPORT` — start hooks run before application setup wires the transports. Defer the first publish; an `@OnStop` **can** publish (transports are torn down after stop hooks).

## Multi-Pod: RedisTransport

By default Asena uses `BunLocalTransport` (`server.publish()` directly — zero overhead, single pod). With multiple pods, messages published on one pod do not reach sockets on another; plug in `RedisTransport` from `@asenajs/asena-redis` via the `@Config` class:

```typescript
import { Config } from '@asenajs/asena/decorators';
import { Inject } from '@asenajs/asena/decorators/ioc';
import { ConfigService } from '@asenajs/ergenecore'; // hono-adapter: same shape
import { RedisTransport } from '@asenajs/asena-redis';

@Config()
export class AppConfig extends ConfigService {
  @Inject('AppRedis')
  private redis: AppRedis;

  public transport() {
    return new RedisTransport(this.redis);           // reuse a @Redis service
    // or: return new RedisTransport({ url: 'redis://localhost:6379' });
  }
}
```

Options: `new RedisTransport(source, { channel: 'asena:ws:transport' })` — `channel` is the Redis pub/sub channel (default shown).

How it works: each instance gets a unique pod ID; a publish is delivered locally (via `server.publish()` for `this.to()`, via the socket's own `ws.publish()` for `socket.publish()` — which is what keeps the sender excluded), relayed over Redis pub/sub with the pod ID, delivered by other pods to their local sockets, and deduplicated for the originating pod. No code changes: rooms, Ulak, and services work identically.

**Warning:** transport `destroy()` is only called from `server.stop()` since 0.10.0 — earlier versions never called it, leaking a Redis subscriber + publisher connection per stop/start cycle (test suites, rolling deploys). Custom transports must make `destroy()` idempotent: the transport object is deliberately reused when the server restarts. Teardown runs under a 5s ceiling with failures logged and skipped.

Custom transports implement `WebSocketTransport` (from `@asenajs/asena`): required `publish(topic, data)` (local `server.publish()` **plus** broker — used by `this.to()`), optional `publishRemote(topic, data)` (broker only, no local delivery — used by `socket.publish()` after Bun's sender-excluding local publish), `init(server)` (before any connection), `destroy()`. Implement `publishRemote()`: without it `socket.publish()` falls back to `publish()` and echoes to the sender (pre-0.10.1 behavior); the adapter warns at startup and the fallback is removed in the next major.
