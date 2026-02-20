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
| `container.v0` / `container.v1` | `Radius.Compute/containers@2025-08-01-preview` | `containers` |
| `redis.server.v0` | `Applications.Datastores/redisCaches@2023-10-01-preview` | `radius` |
| `postgres.server.v0` | `Radius.Data/postgreSqlDatabases@2025-08-01-preview` | `radiusResources` |
| `mysql.server.v0` | `Radius.Data/mySqlDatabases@2025-08-01-preview` | `radiusResources` |
| Application | `Radius.Core/applications@2025-08-01-preview` | `radius` |
| Environment (param) | `Radius.Core/environments@2025-08-01-preview` | `radius` |
| External binding (gateway) | `Radius.Compute/routes@2025-08-01-preview` | `containers` |

**Note**: `Radius.Data/redisCaches` does not yet have a YAML manifest in `resource-types-contrib`. For Redis, the fallback is `Applications.Datastores/redisCaches@2023-10-01-preview`. The mapping table should be configurable to accommodate this transition.

**Rationale**: The project is actively transitioning from `Applications.*` to `Radius.*`. New code should target the new convention. The mapping table design allows updating individual entries as new resource types become available.

**Alternatives considered**:
- Always use legacy `Applications.*` names — rejected because they're being deprecated.
- Auto-detect from installed Radius version — rejected per Principle VII (Simplicity Over Cleverness); the conversion is a static file transformation.

### 3. Bicep Extension Declarations

**Task**: Which `extension` declarations to include in generated Bicep files.

**Decision**: Generate the minimum set of extensions needed for the resource types present in the conversion output. The required extensions are:
- `extension radius` — always included (provides `Radius.Core/applications`, `Radius.Core/environments`)
- `extension containers` — included when container resources are present
- `extension radiusResources` — included when data-store resources (Redis, PostgreSQL, MySQL) are present

**Rationale**: Matches the patterns observed in `resource-types-contrib/Compute/containers/test/app.bicep` and `docs/content/tutorials/deploy-application/snippets/app.bicep`. Only declaring needed extensions keeps the output clean.

**Alternatives considered**:
- Always include all extensions — rejected as it would add unused imports.
- No extensions (rely on Bicep auto-discovery) — rejected; explicit extension declarations are required by the Radius Bicep toolchain.

### 4. Aspire Expression Reference Resolution

**Task**: How to convert Aspire's `{resource.bindings.port.host}` interpolation references to Bicep.

**Decision**: Implement a regex-based expression parser that extracts reference patterns of the form `{resource.property.path}` and resolves them to:
- **Bicep resource references**: `{cache.bindings.tcp.host}` → Bicep property reference on the corresponding resource
- **Connection string references**: `{cache.connectionString}` → triggers a `connections` entry on the consuming container

**Note**: Parameter references (e.g., `{cache-password.value}` → Bicep parameter reference) were identified in the manifest but `parameter.v0` mapping is out of scope for v1. These expression patterns are documented here for future reference.

**Expression patterns observed in sample manifest**:
| Pattern | Example | Maps to |
|---|---|---|
| `{resource.bindings.name.host}` | `{cache.bindings.tcp.host}` | Service discovery / connection |
| `{resource.bindings.name.port}` | `{cache.bindings.tcp.port}` | Connection port reference |
| `{resource.bindings.name.url}` | `{app.bindings.http.url}` | Full URL reference |
| `{resource.bindings.name.targetPort}` | `{app.bindings.http.targetPort}` | Container port ref on self |
| `{resource.value}` | `{cache-password.value}` | Parameter value (out of scope for v1) |
| `{resource.connectionString}` | `{cache.connectionString}` | Full connection string |
| `{resource.inputs.name}` | `{cache-password.inputs.value}` | Input parameter (out of scope for v1) |

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

2. **Mapper tests** (`mapper_test.go`): Table-driven tests validating each Aspire→Radius mapping independently. Tests cover container mapping, binding→port conversion, expression resolution, parameter generation, backing-service mapping, gateway generation for external bindings, and skipping errored resources with appropriate warnings.

3. **Emitter/golden file tests** (`emitter_test.go`): End-to-end tests that parse a full sample manifest, map it, emit Bicep, and compare against golden `.bicep` files in `testdata/`. Uses `testutil.CompareWithGoldenFile` pattern if available, or plain `os.ReadFile + assertEqual`. Includes a golden file test for the `aspire-manifest-invalid-manifest-field.json` manifest to verify errored resources are handled gracefully.

**Rationale**: Follows Principle IV (Testing Pyramid). Table-driven tests are idiomatic Go. Golden file tests catch formatting regressions. No integration tests needed — this is a pure file transformation with no external dependencies.

**Alternatives considered**:
- Functional tests deploying generated Bicep — valuable but belongs in a separate integration test suite, not in unit tests. Deferred to post-merge validation.
