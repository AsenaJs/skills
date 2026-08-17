# OpenAPI

Automatic OpenAPI 3.1 spec generation (`@asenajs/asena-openapi`) from existing validators — zero extra annotations.

## Install & Quickstart

```bash
bun add @asenajs/asena-openapi
```

Requires Bun >= 1.3.12, `@asenajs/asena` >= 0.10.0, zod `^4.3.6`. Zero runtime dependencies (peers: asena, reflect-metadata, zod).

```typescript
import { OpenApi, OpenApiPostProcessor } from '@asenajs/asena-openapi';

@OpenApi({
  info: { title: 'My API', version: '1.0.0' },
  path: '/api/openapi',
  ui: true,
})
export class AppOpenApi extends OpenApiPostProcessor {}
```

Auto-discovered by the IoC container — no registration. Serves `GET /api/openapi` (OpenAPI 3.1 JSON, JSON Schema draft-2020-12) and `GET /api/openapi/ui` (Swagger UI). The PostProcessor intercepts every `@Controller` during IoC setup, extracts every route decorator's metadata (`@Get` … `@Patch`, `@All`), and converts validator Zod schemas via `z.toJSONSchema()` — validators validate requests AND generate the docs.

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

## Options & Swagger UI

| Option | Type | Default | Description |
|---|---|---|---|
| `info` | `{ title, version, description? }` | — | Required API metadata |
| `path` | `string` | `'/openapi'` | Base path for spec and UI endpoints |
| `ui` | `boolean` | `false` | Swagger UI at `{path}/ui` |
| `servers` | `ServerObject[]` | — | e.g. `[{ url: 'https://api.example.com', description: 'Production' }]` |
| `converters` | `SchemaConverter[]` | `[ZodSchemaConverter]` | Pluggable — implement `SchemaConverter` for custom schema types |

**Warning:** Swagger UI loads `swagger-ui-dist@5` from the unpkg CDN — it needs internet access. In air-gapped production, set `ui: false` and use an external docs tool.

Deeper detail: `https://asena.sh/raw/packages/openapi.md`.
