# Research: Aspire Manifest to Bicep Conversion

**Feature**: 001-aspire-to-bicep
**Date**: 2026-02-19

## Research Tasks & Findings

### 1. CLI Command Registration Pattern

**Task**: How to add a new top-level command group (`rad aspire`) to the Radius CLI.

**Decision**: Follow the exact same pattern as `rad bicep` — a thin parent Cobra command file (`aspire.go`) under `cmd/rad/cmd/` registering a `Use: "aspire"` command, then subpackages under `pkg/cli/cmd/aspire/convert/` implementing the `framework.Runner` interface (Validate/Run).

**Rationale**: Every existing command in the Radius CLI uses this pattern. `bicep.go` is the simplest exemplar — a 5-line command group that gets `AddCommand()` for its subcommands in `initSubCommands()`.

**Alternatives considered**:
- Nesting under `rad bicep` — rejected because the command converts *from* Aspire *to* Bicep, not a Bicep-specific operation. The `aspire` namespace also allows future Aspire-related subcommands.
- Standalone binary — rejected because the feature should be part of the `rad` CLI for discoverability and consistent UX.

### 2. Radius Resource Type Naming (Old vs New)

**Task**: Which resource type naming convention to use in generated Bicep output.

**Decision**: Use the **new-style** `Radius.*` resource types with `@2025-08-01-preview` API version as the default target. This aligns with the project's transition direction.

**Findings**:
| Aspire Type | Radius Resource Type | Bicep Extension |
|---|---|---|
| `container.v0` / `container.v1` | `Radius.Compute/containers@2025-08-01-preview` | `radius` |
| `redis.server.v0` | `Applications.Datastores/redisCaches@2023-10-01-preview` | `radius` |
| `postgres.server.v0` | `Radius.Data/postgreSqlDatabases@2025-08-01-preview` | `radius` |
| `mysql.server.v0` | `Radius.Data/mySqlDatabases@2025-08-01-preview` | `radius` |
| Application | `Radius.Core/applications@2025-08-01-preview` | `radius` |
| Environment (param) | `Radius.Core/environments@2025-08-01-preview` | `radius` |
| External binding (gateway) | `Radius.Compute/routes@2025-08-01-preview` | `radius` |

**Note**: `Radius.Data/redisCaches` does not yet have a YAML manifest in `resource-types-contrib`. For Redis, the fallback is `Applications.Datastores/redisCaches@2023-10-01-preview`. The mapping table should be configurable to accommodate this transition.

**Rationale**: The project is actively transitioning from `Applications.*` to `Radius.*`. New code should target the new convention. The mapping table design allows updating individual entries as new resource types become available.

**Alternatives considered**:
- Always use legacy `Applications.*` names — rejected because they're being deprecated.
- Auto-detect from installed Radius version — rejected per Principle VII (Simplicity Over Cleverness); the conversion is a static file transformation.

### 3. Bicep Extension Declarations

**Task**: Which `extension` declarations to include in generated Bicep files.

**Decision**: Generate only `extension radius` at the top of every output file. All Radius resource types (`Radius.Core/applications`, `Radius.Compute/containers`, `Radius.Compute/routes`, `Radius.Data/*`) are available through this single extension.

**Rationale**: Testing with the actual Radius Bicep toolchain confirmed that `extension radius` alone is sufficient for all resource types used in the conversion output. The earlier assumption that `extension containers` and `extension radiusResources` were needed was incorrect — the `radius` extension provides the full Radius type catalog.

**Alternatives considered**:
- Multiple extensions per resource type — rejected after testing showed `extension radius` covers all types. Multiple extensions caused compilation warnings.
- No extensions (rely on Bicep auto-discovery) — rejected; explicit extension declarations are required by the Radius Bicep toolchain.

### 4. Aspire Expression Reference Resolution

**Task**: How to convert Aspire's `{resource.bindings.port.host}` interpolation references to Bicep.

**Decision**: Implement a regex-based expression parser that extracts reference patterns of the form `{resource.property.path}` and resolves them to concrete Bicep constructs based on the referenced resource type:
- **Binding host references**: `{cache.bindings.tcp.host}` → string literal of the resource name (e.g., `'cache'`)
- **Binding port references**: `{cache.bindings.tcp.port}` and `{resource.bindings.name.targetPort}` → string literal of the port number (e.g., `'6379'`); self-references (same resource) resolve to the literal value from that resource's bindings
- **Parameter value references**: `{cache-password.value}` → Bicep parameter reference (e.g., `cache_password`) when the referenced resource is a `parameter.v0` with `secret: true` input
- **Annotated string references**: `{cache-password-uri-encoded.value}` → Bicep variable reference (e.g., `cache_password_uri_encoded`) when the referenced resource is an `annotated.string`
- **Connection string references**: `{cache.connectionString}` → fully expanded by recursively resolving the resource's connection string template, triggers a `connections` entry on the consuming container
- **Composite expressions**: values containing multiple embedded references produce Bicep string interpolation (e.g., `'redis://:${cache_password_uri_encoded}@cache:6379'`)

**Expression patterns observed in sample manifest**:
| Pattern | Example | Maps to |
|---|---|---|
| `{resource.bindings.name.host}` | `{cache.bindings.tcp.host}` | String literal: `'cache'` |
| `{resource.bindings.name.port}` | `{cache.bindings.tcp.port}` | String literal: `'6379'` |
| `{resource.bindings.name.url}` | `{app.bindings.http.url}` | Full URL reference |
| `{resource.bindings.name.targetPort}` | `{app.bindings.http.targetPort}` | String literal of port (self-ref: `'8000'`) |
| `{resource.value}` | `{cache-password.value}` | Bicep parameter reference: `cache_password` |
| `{resource.value}` (annotated.string) | `{cache-password-uri-encoded.value}` | Bicep variable reference: `cache_password_uri_encoded` |
| `{resource.connectionString}` | `{cache.connectionString}` | Fully expanded connection string |
| `{resource.inputs.name}` | `{cache-password.inputs.value}` | Input parameter (handled via parameter mapping) |

**Rationale**: Regex parsing is simple and sufficient for the well-defined Aspire expression format. The expressions are not arbitrary — they follow a predictable `{name.path}` pattern.

**Alternatives considered**:
- Full template engine — rejected per Principle VII; the expression format is constrained.
- String replacement only — rejected because connection strings need to trigger `connections` block generation, not just text substitution.

### 5. Bicep Text Generation Approach

**Task**: How to generate syntactically correct Bicep text from the parsed/mapped data.

**Decision**: Use Go `text/template` with a set of Bicep templates for each resource type. The templates emit well-formatted Bicep with proper indentation. An intermediate representation (IR) struct captures all the mapped data before rendering.

**Rationale**: `text/template` is a Go standard library with no external dependencies. Templates are easy to read, test, and update. The IR provides clean separation between mapping logic and output formatting. The existing `bicep-tools/` pipeline demonstrates that Go-based Bicep generation is viable, though it targets type definitions rather than application files.

**Alternatives considered**:
- String concatenation/builders — rejected because it mixes formatting with logic and is error-prone for indentation.
- AST-based generation using `bicep-types-go` — rejected because the library generates type definitions, not application Bicep files. Over-engineering for template output.

### 6. File Overwrite Safety

**Task**: How to handle output file conflicts (FR-010).

**Decision**: Use `os.Stat()` to check if the target file exists. If it does and `--force` is not set, print an error message to stderr and exit with non-zero code. If `--force` is set, overwrite. Use the `filesystem.FileSystem` interface from `pkg/cli/filesystem` for testability.

**Rationale**: Matches standard CLI conventions. The `filesystem.FileSystem` abstraction already exists in the codebase and supports `os.Stat` equivalent operations. The `generate-kubernetes-manifest` command uses the same pattern for file output.

**Alternatives considered**:
- Interactive prompt ("Overwrite? y/n") — rejected because the `rad` CLI favors non-interactive commands with explicit flags. Prompts complicate scripting and CI pipelines.
- Backup existing file (.bak) — rejected per Principle VII; a simple flag is sufficient.

### 7. Aspire Manifest Error Field Handling

**Task**: How to handle Aspire manifest resource entries that contain an `error` field instead of a `type` field.

**Findings**: The Aspire manifest publisher (`dotnet run --publisher manifest`) sometimes includes resource entries that could not be serialized into the manifest. Instead of the usual resource structure with a `type` field, these entries contain only an `error` field with a human-readable message. Example from `aspire-manifest-invalid-manifest-field.json`:

```json
"docker-hub": {
  "error": "This resource does not support generation in the manifest."
}
```

This occurs for resources like custom Docker registries, unsupported integrations, or resources that don't have a manifest representation. The entry has no `type`, no `bindings`, no `env` — just the error message.

**Decision**: The parser must deserialize the `error` field into an `Error` string field on `AspireResource`. During mapping, resources with a non-empty `Error` field are detected **before** type-based mapping and handled as follows:
1. Skip the resource entirely (do not attempt type resolution or mapping)
2. Emit a warning to stderr: `Warning: resource "docker-hub": manifest error — This resource does not support generation in the manifest.`
3. Add a `BicepComment` to the output: `// Skipped: docker-hub — manifest error: This resource does not support generation in the manifest.`
4. Increment the "skipped" counter in the conversion summary

**Rationale**: This is the most resilient approach — the tool acknowledges the errored resource without failing the entire conversion. Users get clear visibility into what was skipped and why. The behavior is consistent with how unsupported resource types (FR-013) are handled, but uses distinct language ("manifest error" vs "unsupported resource type") so users can differentiate between the two cases.

**Alternatives considered**:
- Fail the entire conversion — rejected because the errored resource is a manifest-publisher issue, not a user error. The remaining resources are perfectly valid and should still be converted.
- Silently ignore — rejected because users should know which resources were not converted. Silent omission could lead to incomplete deployments.

### 8. Testing Strategy

**Task**: Design the testing approach for the conversion tool.

**Decision**: Three layers of unit tests, all runnable via `make test`:

1. **Parser tests** (`manifest_test.go`): Table-driven tests validating JSON parsing of each Aspire resource type. Tests cover valid inputs, missing fields, malformed JSON, unknown types, and errored resource entries (resources with `error` field, no `type`).

2. **Mapper tests** (`mapper_test.go`): Table-driven tests validating each Aspire→Radius mapping independently. Tests cover container mapping, binding→port conversion, expression resolution (including resolution to literals, parameter refs, variable refs, and string interpolation), parameter generation for secret params, variable generation for URI-encoded annotated strings, connection generation, backing-service mapping, gateway generation for external bindings, skipping errored resources with appropriate warnings, and skipping `buildOnly` resources with appropriate warnings.

3. **Emitter/golden file tests** (`emitter_test.go`): End-to-end tests that parse a full sample manifest, map it, emit Bicep, and compare against golden `.bicep` files in `testdata/`. Uses `testutil.CompareWithGoldenFile` pattern if available, or plain `os.ReadFile + assertEqual`. Includes a golden file test for the `aspire-manifest-invalid-manifest-field.json` manifest to verify errored resources and `buildOnly` resources are handled gracefully.

**Rationale**: Follows Principle IV (Testing Pyramid). Table-driven tests are idiomatic Go. Golden file tests catch formatting regressions. No integration tests needed — this is a pure file transformation with no external dependencies.

**Alternatives considered**:
- Functional tests deploying generated Bicep — valuable but belongs in a separate integration test suite, not in unit tests. Deferred to post-merge validation.

### 9. Build-Only Resource Exclusion (`build.buildOnly: true`)

**Task**: How to handle Aspire `container.v1` resources whose `build` configuration contains `"buildOnly": true`.

**Findings**: The Aspire manifest publisher uses `buildOnly: true` on container resources that are build-time-only artifacts. These resources produce files (e.g., compiled static assets) that are consumed by other containers via the `containerFiles` mechanism during image build, but they do not run as independent services at runtime. Example from `aspire-manifest-invalid-manifest-field.json`:

```json
"frontend": {
  "type": "container.v1",
  "build": {
    "context": "frontend",
    "dockerfile": "frontend.Dockerfile",
    "buildOnly": true
  },
  "env": { ... },
  "bindings": { ... }
}
```

In the sample manifest, `frontend` is a build-only container whose output files are consumed by the `app` container via `containerFiles`. The `frontend` container itself should never appear in the Radius deployment because it has no runtime purpose — it only exists to produce static files during the Docker build phase.

**Decision**: Resources with `build.buildOnly: true` MUST be excluded from conversion entirely. The exclusion check occurs **after** error-field detection (FR-018) but **before** normal container mapping (FR-014). The flow is:
1. Check for `error` field → skip per FR-018
2. Check for `build.buildOnly: true` → skip per FR-019
3. Check for `parameter.v0` with `secret: true` → map to `@secure()` param per FR-020
4. Check for `annotated.string` with `filter: "uri"` → map to `uriComponent()` variable per FR-021
5. Check type in mapping table → unsupported warning per FR-013 if not found
6. Map normally if type is supported

When a `buildOnly` resource is detected:
1. Skip the resource (do not generate any Radius resource)
2. Emit a warning: `Warning: resource "frontend" (container.v1): skipped — build-only artifact (build.buildOnly: true)`
3. Add a `BicepComment`: `// Skipped: frontend — build-only artifact (build.buildOnly: true), not a runtime container`
4. Increment the "skipped" counter in the conversion summary

**Rationale**: Build-only containers have no runtime purpose in Radius. Converting them to Radius container resources would create non-functional deployments (the image doesn't expose a service, it only produces build artifacts). Skipping them with a clear warning keeps the generated Bicep clean and accurate while informing the user about what was excluded.

**Alternatives considered**:
- Convert with a warning (match current FR-014 behavior for build containers) — rejected because `buildOnly` containers are fundamentally different from regular build containers. A regular build container still runs as a service; a `buildOnly` container does not.
- Silently ignore — rejected because users should know which resources were excluded. Consistent with the error-field handling decision (Research #7).

### 10. Secret Parameter Handling (`parameter.v0` with `secret: true`)

**Task**: How to handle Aspire `parameter.v0` resources whose inputs include `secret: true`.

**Findings**: The Aspire manifest uses `parameter.v0` resources to represent configurable values. When an input has `secret: true`, it represents a sensitive value (e.g., a database password) that should not be hardcoded. Example from the sample manifests:

```json
"cache-password": {
  "type": "parameter.v0",
  "value": "{cache-password.inputs.value}",
  "inputs": {
    "value": {
      "type": "string",
      "secret": true,
      "default": {
        "generate": { "minLength": 22, "special": false }
      }
    }
  }
}
```

Other resources reference the parameter value via `{cache-password.value}` in their environment variables. The generated Bicep must provide a way for users to supply this secret at deploy time.

**Decision**: Map `parameter.v0` resources with `secret: true` inputs to Bicep `@secure()` parameter declarations. The parameter name is derived from the Aspire resource name with hyphens converted to underscores (e.g., `cache-password` → `cache_password`). Expression references to `{cache-password.value}` resolve to the Bicep parameter name `cache_password`. Non-secret `parameter.v0` resources remain unsupported and emit warnings per FR-013.

The generated Bicep output includes:
```bicep
@secure()
@description('Redis password for the cache container.')
param cache_password string
```

And environment variable references resolve to bare parameter references:
```bicep
env: {
  CACHE_PASSWORD: cache_password
  REDIS_PASSWORD: cache_password
}
```

**Rationale**: Secret parameters are critical for deployable Bicep output. Without them, the generated file cannot be deployed because containers that depend on secrets would have unresolved references. Mapping secret parameters to `@secure()` Bicep params follows Bicep best practices and allows users to supply values via `rad deploy --parameters cache_password=mysecret` or parameter files. The `@secure()` decorator ensures the value is not logged or stored in deployment history.

**Alternatives considered**:
- Leave all parameters as unsupported — rejected because this forces users to manually add `@secure()` params and fix all expression references, defeating the purpose of automated conversion. The working app.bicep confirmed this is needed for a deployable output.
- Map all `parameter.v0` resources (including non-secret) — rejected because non-secret parameters often represent computed or internal values that don't map cleanly to Bicep parameters. Limiting to `secret: true` is the highest-value mapping.
- Generate default values from the `generate` configuration — rejected because secrets should be user-supplied at deploy time, not hardcoded with generated defaults.

### 11. Annotated String Handling (`annotated.string` with `filter: "uri"`)

**Task**: How to handle Aspire `annotated.string` resources with filter transformations.

**Findings**: The Aspire manifest uses `annotated.string` resources to represent derived/transformed values. The `value` field contains an expression reference, and the `filter` field specifies the transformation. Example:

```json
"cache-password-uri-encoded": {
  "type": "annotated.string",
  "value": "{cache-password.value}",
  "filter": "uri"
}
```

This resource represents the URI-encoded version of the `cache-password` parameter, used in constructing `redis://` URIs where the password must be percent-encoded.

**Decision**: Map `annotated.string` resources with `filter: "uri"` to Bicep variable declarations using the `uriComponent()` function. The variable name is derived from the Aspire resource name with hyphens converted to underscores. The source value reference is resolved to the corresponding Bicep parameter or variable name.

The generated Bicep output includes:
```bicep
@description('URI-encoded Redis password (for constructing redis:// URIs).')
var cache_password_uri_encoded = uriComponent(cache_password)
```

Environment variable references containing `{cache-password-uri-encoded.value}` within composite expressions resolve to Bicep string interpolation:
```bicep
CACHE_URI: 'redis://:${cache_password_uri_encoded}@cache:6379'
```

An unsupported comment is still included in the Bicep output (`// Unsupported: cache-password-uri-encoded (annotated.string) — manual configuration required`) because only the `uri` filter is handled and other `annotated.string` resources with different filters would still need manual attention.

**Rationale**: The `uri` filter is the most common `annotated.string` use case in Aspire manifests — URI-encoding passwords for connection string construction. Bicep's built-in `uriComponent()` function provides an exact equivalent. Without this mapping, environment variables referencing the URI-encoded value would contain unresolved expressions, causing deployment failures.

**Alternatives considered**:
- Treat all `annotated.string` as unsupported — rejected because the working app.bicep proved that resolving the `uri` filter to `uriComponent()` is essential for deployable output.
- Support all filter types — rejected per Principle VII (Simplicity Over Cleverness); only `uri` has been observed in real manifests. Other filters can be added as they are encountered.
- Inline the `uriComponent()` call at each usage site — rejected because a named variable is cleaner, avoids repetition, and matches the Aspire manifest's modeling of the value as a named resource.
