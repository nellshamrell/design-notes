# Speckit Prompt: `rad app graph --file` Command

## Feature Request

Add a `--file` flag to the existing `rad app graph` CLI command that generates an application graph directly from a Bicep file, without requiring a deployed Radius environment. This enables developers to preview their application's resource topology and connections at design time, before deploying.

## Context

### Existing Behavior

The `rad app graph <app-name>` command queries a live Radius environment to build an application graph. It calls the `GET /providers/Applications.Core/applications/{name}/getGraph` API endpoint, which fetches deployed resources and uses `computeGraph()` (in `pkg/corerp/frontend/controller/applications/graph_util.go`) to build the graph via breadth-first traversal.

The existing graph output includes:
- Resource nodes (name, type, provisioning state)
- Connections between resources (inbound/outbound edges)
- Output resources (underlying cloud/K8s resources created by each Radius resource)

### What `computeGraph()` Needs

The graph builder operates on `[]generated.GenericResource` objects. For each resource it reads:

- `id` — Full Radius resource ID (e.g., `/planes/radius/local/resourceGroups/default/providers/Applications.Core/containers/frontend`). Used as the graph node key.
- `properties.application` — Which application the resource belongs to.
- `properties.connections` — Map of named outbound connections, each with a `source` field (a resource ID or URL like `http://backend:3000`).
- `properties.routes` — Array of gateway route objects, each with a `destination` field.
- `properties.provisioningState` — Deployment status (only available after deployment).
- `properties.status.outputResources` — Cloud/K8s sub-resources created (only available after deployment).

Connection sources are resolved by `findSourceResource()`, which tries: (1) parsing as a resource ID, (2) parsing as a URL and matching the hostname against resource names in the resource list.

### What's Available in a Bicep File

A Bicep file compiles (via `rad-bicep build --stdout`) to an ARM JSON template with this structure:

```json
{
  "resources": {
    "symbolicName": {
      "type": "Applications.Core/containers@2023-10-01-preview",
      "properties": {
        "name": "frontend",
        "properties": {
          "application": "[reference('app').id]",
          "connections": {
            "backend": { "source": "http://backend:3000" }
          },
          "container": { ... }
        }
      }
    }
  }
}
```

Note the double-nesting: the outer `properties` is the ARM resource envelope; the inner `properties` contains the Radius resource properties (connections, routes, application, etc.).

Key data availability from a Bicep file:
- ✅ Resource types and names — explicitly declared
- ✅ Connections with literal URL sources (e.g., `http://backend:3000`) — directly extractable
- ✅ Connections with resource ID references (e.g., `otherResource.id` in Bicep → `[reference('otherResource').id]` in ARM JSON) — resolvable by looking up the symbolic name in the same template
- ✅ Gateway routes with destinations — directly extractable
- ❌ Provisioning state — not available (not yet deployed)
- ❌ Output resources — not available (created at deploy time)
- ⚠️ Connections using parameter values or `format()` expressions — not resolvable without parameter values

### Existing Code to Reuse

- `PrepareTemplate()` in `pkg/cli/bicep/types.go` — compiles `.bicep` → ARM JSON `map[string]any`
- `computeGraph()` in `pkg/corerp/frontend/controller/applications/graph_util.go` — builds graph from `[]GenericResource`
- `findSourceResource()` — resolves connection source strings to resource IDs
- `ApplicationGraphResponse` / `ApplicationGraphResource` / `ApplicationGraphConnection` types in `pkg/corerp/api/v20231001preview/`

## Proposed Command

```bash
# Static graph from a Bicep file (new)
rad app graph --file app.bicep

# Live graph from deployed app (existing, unchanged)
rad app graph myapp
```

The `--file` flag and the positional `<app-name>` argument should be mutually exclusive.

## Requirements

### Functional Requirements

1. **Bicep compilation**: Accept a `.bicep` or `.json` (pre-compiled ARM template) file path via the `--file` flag. Use the existing `PrepareTemplate()` function to compile Bicep to ARM JSON.

2. **Resource extraction**: Walk the `resources` map in the compiled ARM JSON template. For each resource, extract:
   - Resource type (strip the `@api-version` suffix)
   - Resource name (from `properties.name`)
   - Radius properties (from `properties.properties` — the inner properties)

3. **Synthetic resource ID generation**: Construct a full Radius resource ID for each resource using the pattern:
   ```
   /planes/radius/local/resourceGroups/default/providers/{Type}/{Name}
   ```
   Use a default placeholder resource group since the graph only needs unique IDs as keys.

4. **Connection and route extraction**: Extract `connections` and `routes` from each resource's Radius properties (the inner `properties`), exactly as `computeGraph()` does for live resources.

5. **ARM expression resolution for connections**: When a connection source or route destination is an ARM expression:
   - `[reference('symbolicName').id]` → look up `symbolicName` in the template's resources, resolve to its synthesized resource ID.
   - Literal strings (URLs like `http://backend:3000`) → pass through as-is (the existing `findSourceResource()` handles hostname-to-resource matching).
   - Unresolvable expressions (parameters, format calls) → skip the connection with a warning, or include a placeholder indicating the connection target is unknown.

6. **Graph computation**: Convert extracted resources to `[]generated.GenericResource` and call the existing `computeGraph()` function (or an equivalent adapted for static use).

7. **Application scoping**: All resources in the Bicep file that have an `application` property referencing an application resource in the same file should be treated as application-scoped. If the file contains exactly one application resource, all resources referencing it are included. If no application resource is found, treat all resources as part of a single implicit application.

8. **Output format**: Use the same output format as the existing `rad app graph` command (text tree with connections). Set `provisioningState` to `"NotDeployed"` and `outputResources` to an empty list for all resources.

9. **Warnings**: Emit warnings (to stderr) for:
   - Connection sources that could not be resolved (ARM expressions with parameters, etc.)
   - Conditional resources (`if` in Bicep) — include them but warn they may not be deployed
   - Module references — resources inside Bicep modules are compiled as nested deployments and may not be visible

### Non-Functional Requirements

10. **No Radius environment required**: The `--file` flag must work completely offline. No API calls to a Radius control plane. The Bicep compiler (`rad-bicep`) must be available locally (auto-download if needed, matching existing behavior).

11. **Error handling**: Follow the existing `computeGraph()` philosophy of graceful degradation — return partial results rather than failing entirely on unresolvable data.

12. **Deterministic output**: All resources and connections must be sorted for stable, reproducible output (matching existing behavior).

## Scope Boundaries

### In Scope
- Single Bicep files with inline resources
- Pre-compiled ARM JSON template files
- Literal URL connection sources
- `[reference('X').id]` expression resolution within the same template
- All Radius resource types (core + portable)

### Out of Scope (Future Work)
- Recursive processing of Bicep modules / nested ARM templates
- Full ARM template expression evaluation
- Resolving recipe-provisioned output resources
- Comparing static graph vs. live deployed graph (diff mode)
- Parameter file support (`--parameters`)

## Example

Given `app.bicep`:
```bicep
extension radius

param environment string

resource app 'Applications.Core/applications@2023-10-01-preview' = {
  name: 'myapp'
  properties: {
    environment: environment
  }
}

resource frontend 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'frontend'
  properties: {
    application: app.id
    container: {
      image: 'frontend:latest'
      ports: {
        web: { containerPort: 3000 }
      }
    }
    connections: {
      backend: {
        source: 'http://backend:8080'
      }
    }
  }
}

resource backend 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'backend'
  properties: {
    application: app.id
    container: {
      image: 'backend:latest'
      ports: {
        api: { containerPort: 8080 }
      }
    }
    connections: {
      db: {
        source: redis.id
      }
    }
  }
}

resource redis 'Applications.Datastores/redisCaches@2023-10-01-preview' = {
  name: 'redis'
  properties: {
    application: app.id
    environment: environment
  }
}
```

Running `rad app graph --file app.bicep` should produce output like:

```
Displaying application: myapp

Name: frontend (Applications.Core/containers)
Connections:
  frontend -> backend (Outbound)
  frontend <- backend (Inbound from gateway, if present)

Name: backend (Applications.Core/containers)
Connections:
  backend -> redis (Outbound)
  backend <- frontend (Inbound)

Name: redis (Applications.Datastores/redisCaches)
Connections:
  redis <- backend (Inbound)
```

(Exact output format should match the existing `rad app graph` text rendering.)

## Key Implementation Details

### ARM JSON Double-Nesting

In compiled ARM JSON, Radius resource properties are nested under `properties.properties`:
```
template["resources"]["symbolicName"]["properties"]       ← ARM envelope (has "name")
template["resources"]["symbolicName"]["properties"]["properties"]  ← Radius properties (has "connections", "routes", "application")
```

### Resource Type Parsing

ARM JSON resource types include the API version: `Applications.Core/containers@2023-10-01-preview`. Strip the `@...` suffix to get the Radius resource type: `Applications.Core/containers`.

### ARM Reference Resolution

The pattern `[reference('symbolicName').id]` appears when Bicep compiles `otherResource.id`. To resolve:
1. Parse the expression to extract `symbolicName`
2. Look up `symbolicName` in `template["resources"]`
3. Use that resource's type and name to synthesize its resource ID

A regex like `\[reference\('(\w+)'\)\.id\]` would handle the common case.
