# Reading OpenAPI 3.x and Swagger 2.0 specs

Both versions describe the same things in different places. Check the top-level key first: `"openapi": "3.0.x"` / `"3.1.x"` means OpenAPI 3; `"swagger": "2.0"` means Swagger 2.0. YAML and JSON have the same structure.

## Where things are

| What | OpenAPI 3.x | Swagger 2.0 |
|---|---|---|
| Base URL | `servers[].url` (may contain `{variables}` with defaults under `servers[].variables`) | `schemes[0]` + `://` + `host` + `basePath` |
| Operations | `paths.<path>.<method>` | `paths.<path>.<method>` |
| Path/query/header params | `parameters[]` with `in: path/query/header`, type under `schema` | `parameters[]` with `in: path/query/header`, `type` directly on the parameter |
| Request body | `requestBody.content["application/json"].schema` | a parameter with `in: body` and a `schema` |
| Form data | `requestBody.content["application/x-www-form-urlencoded"]` | parameters with `in: formData` |
| Reusable schemas | `components.schemas` | `definitions` |
| Security schemes | `components.securitySchemes` | `securityDefinitions` |
| Success responses | `responses` keys `200`–`299` | same |

Parameters can be declared on the path item (shared by all methods on that path) as well as on the operation. Merge both, with the operation's version winning.

Resolve `$ref` values by following them to `components.schemas` / `definitions` (or `components.parameters`, `components.requestBodies`). Watch for circular references and stop after a couple of levels.

## Reading large specs

For big specs, don't read the whole file into context at once. Search it (Grep) for `paths`, then the individual operations the user selected, then the schemas those operations reference.

## Example values, in order of preference

1. `example` on the parameter or property
2. `examples` (take the first one's `value`)
3. `default`
4. First entry of `enum`
5. Generated from the schema (see below), marked with a `// TODO: generated value` comment

## Generating values from a schema

| Schema | Generated value |
|---|---|
| `string` | `"test"` |
| `string`, `format: email` | `$"user{Guid.NewGuid():N}@example.com"` (unique per run) |
| `string`, `format: uuid` | `Guid.NewGuid().ToString()` |
| `string`, `format: date` / `date-time` | today's date / current time in ISO 8601 |
| `string` with `minLength`/`maxLength` | a string that fits the limits |
| `integer` / `number` | `minimum` if given, otherwise `1` |
| `boolean` | `true` |
| `array` | a one-item array of a generated item |
| `object` | an object with all `required` properties filled; include optional ones only if they have examples |

## Finding the id to chain between steps

After a create step (usually `POST`), look at its success response schema for the id field: a property named `id`, or `<resource>Id` matching the path parameter name used by later steps (for example `petId` for `/pets/{petId}`). If the create response has no body or no id field, don't chain. Use an example or generated value for the path parameter and mark it with a TODO.
