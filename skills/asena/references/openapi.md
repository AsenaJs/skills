# OpenAPI

Automatic OpenAPI 3.1 spec generation (`@asenajs/asena-openapi` 3.2.x) from existing validators and route context contracts — zero extra annotations.

## Install & Quickstart

```bash
bun add @asenajs/asena-openapi
```

Requires Bun >= 1.4.0, `@asenajs/asena` `^0.12.0`, zod `^4.3.6`. Zero runtime dependencies (peers: asena, reflect-metadata, zod).

```typescript
import { OpenApi, OpenApiPostProcessor } from '@asenajs/asena-openapi';

@OpenApi({
  info: { title: 'My API', version: '1.0.0' },
  path: '/api/openapi',
  ui: 'scalar', // or true / 'swagger'
})
export class AppOpenApi extends OpenApiPostProcessor {}
```

Auto-discovered by the IoC container — no registration. Serves `GET /api/openapi` (OpenAPI 3.1 JSON, JSON Schema draft-2020-12) and `GET /api/openapi/ui` (Swagger UI or Scalar API Reference). The PostProcessor intercepts every `@Controller` during IoC setup, extracts every route decorator's metadata (`@Get` … `@Patch`, `@All`), and converts validator Zod schemas via `z.toJSONSchema()` — validators validate requests AND generate the docs.

## Validator → Spec Mapping

Each `ValidationService` method maps to a spec section:

| Validator method | OpenAPI output | Location |
|---|---|---|
| `json()` | RequestBody | `application/json` |
| `form()` | RequestBody | `multipart/form-data` |
| `query()` | ParameterObject[] | `in: query` |
| `param()` | ParameterObject[] | `in: path` |
| `header()` | ParameterObject[] | `in: header` |
| `response()` | ResponseObject | by status code |

```typescript
@Middleware({ validator: true })
export class CreateUserValidator extends ValidationService {
  json()  { return z.object({ name: z.string().min(1), email: z.string().email() }); }
  query() { return z.object({ page: z.coerce.number().optional() }); }
  param() { return z.object({ id: z.string().uuid() }); }
  response() {
    return {
      201: z.object({ id: z.string(), name: z.string() }),                              // simple form
      400: { schema: z.object({ error: z.string() }), description: 'Validation error' }, // detailed form
    };
  }
}
```

## Route Context Contract → Spec Mapping

A route whose first parameter is a `Context.define()` subclass is documented from the contract — no validator class needed:

| Contract key | OpenAPI output | Location |
|---|---|---|
| `body` | RequestBody (required) | `application/json` |
| `form` | RequestBody (required) | `multipart/form-data` |
| `query` | ParameterObject[] | `in: query` |
| `params` | ParameterObject[] | `in: path` |
| `headers` | ParameterObject[] | `in: header` |

```typescript
import { Context } from '@asenajs/ergenecore'; // or '@asenajs/hono-adapter'
import { z } from 'zod';

class ListOrdersContext extends Context.define({
  query: z.object({ page: z.coerce.number().int().default(1) }),
  params: z.object({ shopId: z.string() }),
}) {}

@Get('/:shopId/orders')
async list(context: ListOrdersContext) { /* ... */ }
```

Contracts carry no response schemas — for documented responses use a validator class with `response()`.

**Input vs output shape:** request schemas (contract targets and the validator's `json()`/`form()`/`query()`/`param()`/`header()`) are documented in their **input** shape — what the client sends; `response()` schemas use the **output** shape. So `.default()` makes a field optional in a request but required in a response, and `z.string().pipe(z.coerce.number())` is a `string` in a request, a `number` in a response. A `.transform()` output has no JSON Schema and becomes `{}`.

**Contracts must convert:** when a contract target cannot be converted to JSON Schema, or a `query`/`params`/`headers` schema is not an object with properties, spec generation **fails** naming the route and target (no partial spec is cached). Contracts may use any Standard Schema library at runtime, but OpenAPI needs a converter for it — `ZodSchemaConverter` covers Zod.

## @Hidden

Exclude routes from the spec at class or method level:

```typescript
import { Hidden } from '@asenajs/asena-openapi';

@Hidden()                 // hides the whole controller
@Controller('/internal')
export class InternalController { /* ... */ }

@Controller('/api')
export class ApiController {
  @Hidden()               // hides just this route
  @Get('/health')
  healthCheck() {}
}
```

## Options & docs UI

| Option | Type | Default | Description |
|---|---|---|---|
| `info` | `{ title, version, description? }` | — | Required API metadata |
| `path` | `string` | `'/openapi'` | Base path for spec and UI endpoints |
| `ui` | `boolean \| 'swagger' \| 'scalar' \| { provider, configuration? }` | — | Docs UI at `{path}/ui`. `true` / `'swagger'` = Swagger UI (`swagger-ui-dist@5`, unpkg); `'scalar'` = Scalar API Reference (`@scalar/api-reference@1`, jsdelivr). Object form passes `configuration` straight into `SwaggerUIBundle` / `Scalar.createApiReference`; its keys override the defaults, `url` included. Unknown provider → error at boot |
| `servers` | `ServerObject[]` | — | e.g. `[{ url: 'https://api.example.com', description: 'Production' }]` |
| `converters` | `SchemaConverter[]` | `[ZodSchemaConverter]` | Pluggable — implement `SchemaConverter` for custom schema types |

### Custom converters

The first converter whose `canConvert()` returns `true` handles a schema. Keep `ZodSchemaConverter` in the list when adding one:

```typescript
import type { JsonSchema, SchemaConverter, SchemaDirection } from '@asenajs/asena-openapi';

export class MySchemaConverter implements SchemaConverter {
  canConvert(schema: unknown, direction?: SchemaDirection): boolean {
    return isMySchema(schema);
  }

  convert(schema: unknown, direction: SchemaDirection = 'output', options?: { strict?: boolean }): JsonSchema | undefined {
    return toJsonSchema(schema, direction);
  }
}
```

`direction` is `'input'` for request schemas, `'output'` for responses. `options.strict` is `true` for contracts: throw or return `undefined` rather than approximate — either fails spec generation with the route and target named.

**Warning:** both UIs load from a CDN — they need internet access. In air-gapped production, leave `ui` unset and use an external docs tool.

## Descriptions

Docs text comes from code you already write: `@Controller({ path, description })` → tag description; `@Get({ path, summary, description })` (any verb decorator) → operation summary/description; `.describe()` on a `query()`/`param()`/`header()` field → parameter description, on a field inside a body/response schema → property description; `.describe()` on the `json()`/`form()` object itself → `requestBody.description` (the schema does not carry it, so it renders once); `response()` detailed form → response description.

Deeper detail: `https://asena.sh/raw/packages/openapi.md`.
