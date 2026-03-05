# Contract: Graph Import & Transformation API

**Date**: 2026-03-05
**Feature**: [spec.md](../spec.md) | [data-model.md](../data-model.md)

## Overview

This contract defines the interfaces for importing, validating, and transforming application graph data in the Radius Dashboard. These are **client-side TypeScript interfaces** — no server API is involved.

## Input Schema: ApplicationGraphResponse

The JSON format produced by `rad app graph --file --output json`:

```typescript
/**
 * The input format from the Radius CLI.
 * Users paste or upload this JSON into the dashboard.
 */
interface ApplicationGraphResponse {
  resources: ApplicationGraphResource[];
}

interface ApplicationGraphResource {
  id: string;
  type: string;
  name: string;
  provisioningState: string;
  outputResources: ApplicationGraphOutputResource[];
  connections: ApplicationGraphConnection[];
}

interface ApplicationGraphOutputResource {
  id: string;
  type: string;
  name: string;
}

interface ApplicationGraphConnection {
  id: string;
  direction: 'Outbound' | 'Inbound';
}
```

## Validation Contract

### Function: `validateApplicationGraphResponse`

```typescript
/**
 * Validates raw parsed JSON against the ApplicationGraphResponse schema.
 *
 * @param data - Parsed JSON object (result of JSON.parse)
 * @returns Validation result with typed data or error messages
 */
function validateApplicationGraphResponse(
  data: unknown
): ValidationResult<ApplicationGraphResponse>;

interface ValidationResult<T> {
  success: boolean;
  data?: T;         // Present when success === true
  errors?: string[]; // Present when success === false; human-readable messages
}
```

### Validation rules

| ID | Rule | Error message template |
|----|------|----------------------|
| V-001 | Input must be a JSON object | `"Expected a JSON object, got {typeof input}"` |
| V-002 | `resources` must be an array | `"Missing or invalid 'resources' field: expected an array"` |
| V-003 | Each resource `id` must be non-empty string | `"Resource at index {i}: 'id' is required and must be a non-empty string"` |
| V-004 | Each resource `type` must be non-empty string | `"Resource at index {i}: 'type' is required and must be a non-empty string"` |
| V-005 | Each resource `name` must be non-empty string | `"Resource at index {i}: 'name' is required and must be a non-empty string"` |
| V-006 | Each resource `provisioningState` must be a string | `"Resource at index {i}: 'provisioningState' must be a string"` |
| V-007 | Each resource `outputResources` must be an array | `"Resource at index {i}: 'outputResources' must be an array"` |
| V-008 | Each resource `connections` must be an array | `"Resource at index {i}: 'connections' must be an array"` |
| V-009 | Each connection `id` must be non-empty string | `"Resource '{name}', connection at index {j}: 'id' is required"` |
| V-010 | Each connection `direction` must be `Outbound` or `Inbound` | `"Resource '{name}', connection at index {j}: 'direction' must be 'Outbound' or 'Inbound'"` |

### Behavior

- Reports **all** validation errors, not just the first — enables users to fix multiple issues at once.
- An empty `resources: []` array passes validation (empty graph is valid).

## Transformation Contract

### Function: `transformToAppGraph`

```typescript
/**
 * Transforms validated ApplicationGraphResponse into the dashboard's AppGraph type.
 * 
 * @param response - Validated ApplicationGraphResponse
 * @param name - Optional application name (defaults to "Preview")
 * @returns AppGraph suitable for the <AppGraph> component
 */
function transformToAppGraph(
  response: ApplicationGraphResponse,
  name?: string
): AppGraph;
```

### Transformation rules

| Input field | Output field | Transformation |
|-------------|-------------|---------------|
| _(none)_ | `AppGraph.name` | Use `name` param or `"Preview"` |
| `resource.id` | `Resource.id` | Direct copy |
| `resource.name` | `Resource.name` | Direct copy |
| `resource.type` | `Resource.type` | Direct copy |
| `resource.type` | `Resource.provider` | Extract namespace: `type.split('/')[0]` |
| `resource.provisioningState` | `Resource.provisioningState` | Direct copy |
| `resource.outputResources` | `Resource.resources` | Map each to `Resource` type |
| `resource.connections` | `Resource.connections` | Enrich with looked-up name, type, provider |
| `connection.id` | `Connection.id` | Direct copy |
| `connection.direction` | `Connection.direction` | Direct copy |
| _(lookup by connection.id)_ | `Connection.name` | Find resource by ID, use its `name` |
| _(lookup by connection.id)_ | `Connection.type` | Find resource by ID, use its `type` |
| _(derive from looked-up type)_ | `Connection.provider` | `type.split('/')[0]` |

### Edge cases

- If `connection.id` references a resource not in the graph, use the last segment of the ID as `name`, `"Unknown"` as `type`, and `"Unknown"` as `provider`.
- `outputResources` mapped to `Resource` type with `provider` derived, `provisioningState: ""`, empty `resources` and `connections`.

## JSON Parse Contract

### Function: `parseGraphJson`

```typescript
/**
 * Parses raw text as JSON and validates as ApplicationGraphResponse.
 * Combines JSON.parse + validateApplicationGraphResponse.
 *
 * @param text - Raw JSON text (from paste or file read)
 * @returns Validation result with parsed and validated data or error messages
 */
function parseGraphJson(
  text: string
): ValidationResult<ApplicationGraphResponse>;
```

### Behavior

- If `JSON.parse` throws, return `{ success: false, errors: ["Invalid JSON: {parse error message}"] }`
- If parse succeeds, delegate to `validateApplicationGraphResponse`
