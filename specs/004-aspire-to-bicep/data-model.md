# Data Model: Aspire-to-Bicep PoC

**Branch**: `004-aspire-to-bicep` | **Date**: 2026-02-23

## Entity Overview

The conversion pipeline has two model layers: the **Parsed Model** (representation of input artifacts — YAML templates and Bicep files) and the **Radius Model** (representation of the target app.bicep output). The mapper transforms one into the other, and the reporter tracks the lineage between them.

## Parsed Model (Input)

These entities represent the structure extracted from `azd infra synth` artifacts. Per R-001, the real output consists of per-service YAML templates (`.tmpl.yaml`) in the AppHost's `infra/` directory and solution-level Bicep files in the top-level `infra/` directory.

### AspireAppDescriptor

Top-level entity representing the fully parsed Aspire application.

| Field | Type | Description |
|-------|------|-------------|
| `RootDir` | `string` | Absolute path to the Aspire application root directory |
| `AppHostDir` | `string` | Absolute path to the AppHost project directory (contains `infra/`) |
| `ServiceTemplates` | `[]ServiceTemplate` | Parsed per-service YAML templates |
| `MainBicep` | `*BicepFile` | Parsed solution-level `main.bicep` (parameters, modules) |
| `ParametersJSON` | `map[string]any` | Parsed `main.parameters.json` content |

### ServiceTemplate

Represents a parsed per-service `.tmpl.yaml` file.

| Field | Type | Description |
|-------|------|-------------|
| `Path` | `string` | Absolute file path of the source `.tmpl.yaml` file |
| `ServiceName` | `string` | Service name from `tags.aspire-resource-name` or filename stem |
| `AzdServiceName` | `string` | Service name from `tags.azd-service-name` |
| `Ingress` | `*IngressConfig` | Parsed ingress configuration (port, external, transport) |
| `Containers` | `[]ContainerDef` | Container definitions from `template.containers` |
| `Secrets` | `[]SecretDef` | Secrets from `configuration.secrets` |
| `Tags` | `map[string]string` | Resource tags |

### IngressConfig

Represents the ingress configuration extracted from a `.tmpl.yaml` file.

| Field | Type | Description |
|-------|------|-------------|
| `External` | `bool` | Whether the service is externally accessible |
| `TargetPort` | `int` | Target port number (extracted from `targetPort` or `{{ targetPortOrDefault N }}`) |
| `Transport` | `string` | Transport protocol (`http`, `tcp`) |

### ContainerDef

Represents a container definition from `template.containers[]`.

| Field | Type | Description |
|-------|------|-------------|
| `Image` | `string` | Image expression (typically `{{ .Image }}` → placeholder) |
| `Name` | `string` | Container name |
| `Env` | `[]EnvVar` | Environment variables |
| `Command` | `[]string` | Container command override (if any) |
| `Args` | `[]string` | Container arguments (if any) |

### EnvVar

Represents an environment variable from a container definition.

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Environment variable name |
| `Value` | `string` | Literal value (if set directly) |
| `SecretRef` | `string` | Secret reference name (if set via `secretRef`) |

### SecretDef

Represents a secret from `configuration.secrets[]`.

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Secret name |
| `Value` | `string` | Secret value expression (may contain Go template expressions) |

### BicepFile

Represents the solution-level `main.bicep` file (used for application-level context only).

| Field | Type | Description |
|-------|------|-------------|
| `Path` | `string` | Absolute file path of the source Bicep file |
| `Resources` | `[]BicepResource` | Resource declarations found in the file |
| `Parameters` | `[]BicepParameter` | Parameter declarations found in the file |
| `Modules` | `[]BicepModule` | Module declarations found in the file |
| `Variables` | `[]BicepVariable` | Variable declarations found in the file |

### BicepResource

Represents a `resource` declaration in Bicep.

| Field | Type | Description |
|-------|------|-------------|
| `SymbolicName` | `string` | Bicep symbolic name (e.g., `apiservice`) |
| `Type` | `string` | Resource type (e.g., `Microsoft.App/containerApps@2024-03-01`) |
| `Name` | `string` | Resource name expression |
| `Properties` | `map[string]any` | Nested property tree extracted from the resource body |
| `SourceFile` | `string` | File this resource was extracted from |
| `StartLine` | `int` | Line number where the resource declaration begins |

### BicepParameter

Represents a `param` declaration in Bicep.

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Parameter name |
| `Type` | `string` | Parameter type (e.g., `string`, `int`) |
| `DefaultValue` | `string` | Default value expression, if any |
| `IsSecure` | `bool` | Whether the parameter has `@secure()` decorator |
| `Description` | `string` | `@description()` value, if any |
| `SourceFile` | `string` | File this parameter was extracted from |

### BicepModule

Represents a `module` declaration in `main.bicep`.

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Module symbolic name |
| `Source` | `string` | Module source path (relative to main.bicep) |
| `Parameters` | `map[string]string` | Parameter expressions passed to the module |
| `DependsOn` | `[]string` | Symbolic names this module depends on |

### BicepVariable

Represents a `var` declaration in Bicep.

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Variable name |
| `Value` | `string` | Variable value expression |

## Radius Model (Output)

These entities represent the target Radius application structure to be rendered as app.bicep.

### RadiusApplication

Top-level entity representing the entire application conversion.

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Application name derived from Aspire project name | Directory name or `main.bicep` → `environmentName` param |
| `Containers` | `[]RadiusContainer` | Service containers in the application | Per-service `.tmpl.yaml` templates (services only) |
| `Dependencies` | `[]RadiusDependency` | Infrastructure dependencies (Redis, SQL Server, etc.) | Per-dependency `.tmpl.yaml` templates |
| `Parameters` | `[]RadiusParameter` | Bicep parameters for the output file | Derived from image refs, secrets |
| `Variables` | `[]RadiusVariable` | Bicep variables for the output file | Derived from computed values |

### RadiusContainer

Represents a `Applications.Core/containers` resource.

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Container resource name | `.tmpl.yaml` → `tags.aspire-resource-name` or container `name` |
| `ImageParam` | `string` | Bicep parameter name for the image | Derived: `{name}Image` |
| `ImageDefault` | `string` | Default image value | Always set to `IMAGE_PLACEHOLDER`. The developer must provide actual image references at deploy time via `rad deploy --parameters`. |
| `Ports` | `[]RadiusPort` | Port definitions | `.tmpl.yaml` → `configuration.ingress` |
| `EnvVars` | `map[string]EnvVarValue` | Environment variables | `.tmpl.yaml` → `template.containers[0].env[]`. Plain values use `{ value: '<literal>' }` syntax. Dependency-backed env vars are transformed to Bicep resource expressions (e.g., `ConnectionStrings__weatherdb` → `{ value: sqlserver.listSecrets().connectionString }`, `CACHE_HOST` → `{ value: cache.properties.host }`, `WEATHERDB_PORT` → `{ value: string(sqlserver.properties.port) }`) |
| `Connections` | `[]RadiusConnection` | Connections to other resources | Derived from `ConnectionStrings__*` and `services__*` env vars |
| `IsExternal` | `bool` | Whether this service has external ingress | `.tmpl.yaml` → `ingress.external` |
| `Command` | `[]string` | Container command override | `.tmpl.yaml` → `template.containers[0].command` |

### RadiusPort

Represents a port definition on a container.

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Port name (e.g., `http`, `tcp`) | Derived from `ingress.transport` in `.tmpl.yaml` |
| `ContainerPort` | `int` | Port number | `.tmpl.yaml` → `ingress.targetPort` or `{{ targetPortOrDefault N }}` |
| `Protocol` | `string` | Protocol (`TCP`, `UDP`) | `.tmpl.yaml` → `ingress.transport` |
| `IsPlaceholder` | `bool` | Whether this port is a placeholder (not found in source) | Gap detection |

### RadiusConnection

Represents a connection from one resource to another.

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Connection name (e.g., `cache`, `apiservice`) | Derived from `ConnectionStrings__` suffix or `services__` prefix |
| `TargetResourceName` | `string` | Symbolic name of the target resource | Parsed from env var name pattern |
| `Source` | `string` | Bicep expression for the connection source (e.g., `cache.id`) | Mapper output |

### RadiusDependency

Represents a portable resource (e.g., Redis) or a placeholder for unsupported types.

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Resource name | `.tmpl.yaml` → `tags.aspire-resource-name` |
| `Type` | `string` | Radius resource type (e.g., `Applications.Datastores/redisCaches`, `Applications.Datastores/sqlDatabases`) or empty for placeholders | Mapped from container image/port heuristics |
| `IsRecipeBacked` | `bool` | Whether provisioned by Recipe | `true` for supported types |
| `IsPlaceholder` | `bool` | Whether this is a placeholder (unsupported type) | Gap detection |
| `PlaceholderComment` | `string` | Comment explaining the placeholder | Reporter output |

### RadiusParameter

Represents a `param` declaration in the output app.bicep.

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Parameter name |
| `Type` | `string` | Bicep type (`string`, `int`) |
| `DefaultValue` | `string` | Default value |
| `IsSecure` | `bool` | Whether to add `@secure()` |
| `Description` | `string` | `@description()` text |

### RadiusVariable

Represents a `var` declaration in the output app.bicep.

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Variable name |
| `Value` | `string` | Variable expression |

## Mapping Report Model

These entities track the lineage from source to output for the mapping report.

### MappingEntry

Tracks one field's lineage from source to output.

| Field | Type | Description |
|-------|------|-------------|
| `TargetResource` | `string` | Name of the Radius resource in app.bicep |
| `TargetField` | `string` | Dotted path of the field (e.g., `container.ports.http.containerPort`) |
| `SourceFile` | `string` | Source Bicep file path |
| `SourceField` | `string` | Source field path (e.g., `configuration.ingress.targetPort`) |
| `Value` | `string` | The mapped value |
| `IsGap` | `bool` | Whether this is a gap (field could not be populated) |
| `IsAssumption` | `bool` | Whether a default/assumption was used |
| `GapMessage` | `string` | Explanation of the gap or assumption |

### MappingReport

Aggregates all mapping entries for output.

| Field | Type | Description |
|-------|------|-------------|
| `SourceDirectory` | `string` | Input directory path |
| `GeneratedAt` | `time.Time` | Timestamp of conversion |
| `Entries` | `[]MappingEntry` | All mapping entries |
| `Gaps` | `[]MappingEntry` | Filtered: only gap entries |
| `Assumptions` | `[]MappingEntry` | Filtered: only assumption entries |

## State Transitions

The conversion is stateless — there are no state transitions. The pipeline is:

```
Aspire app directory (azd infra synth output)
    → [Parser] → AspireAppDescriptor (Parsed Model: ServiceTemplates + MainBicep)
    → [Mapper] → RadiusApplication + MappingReport (Radius Model + Lineage)
    → [Generator] → app.bicep (output file)
    → [Reporter] → mapping-report.md + console output
```

## Determinism (FR-012)

To guarantee byte-for-byte identical output on re-runs (FR-012, SC-006, User Story 3):

| Concern | Rule |
|---------|------|
| **Slice ordering** | `RadiusApplication.Containers`, `RadiusApplication.Dependencies`, and `RadiusApplication.Parameters` MUST be sorted lexicographically by `Name` before rendering. |
| **Map iteration** | `RadiusContainer.EnvVars` (Go `map[string]string`) MUST be iterated in sorted key order during template rendering. |
| **MappingReport entries** | `MappingReport.Entries` MUST be sorted by `(TargetResource, TargetField)` before rendering. `Gaps` and `Assumptions` follow the same sort. |
| **Timestamp in output** | The `Date:` header comment in `app.bicep` and `Generated:` field in `mapping-report.md` MUST be omitted from idempotency comparisons. Implementations SHOULD provide a `--deterministic` flag (or equivalent) that replaces the timestamp with a fixed sentinel value for CI/testing use. |
| **File discovery order** | When scanning the input directory, files MUST be processed in lexicographic path order (`filepath.Walk` provides this in Go). |

## Validation Rules

| Entity | Rule | Error Behavior |
|--------|------|----------------|
| Input directory | Must exist and contain an AppHost project with `infra/` directory containing `.tmpl.yaml` files | Fail with FR-010 error message |
| Input directory | Must contain a solution-level `infra/main.bicep` | Fail with FR-010 error message |
| ServiceTemplate | Must have `tags.aspire-resource-name` or a parseable filename | Skip template, log as gap |
| Service template | Must have `template.containers` array | Log port/image as gaps, generate placeholder |
| Redis dependency | Identified by port 6379/tcp transport or `redis` in image/name | Map to `Applications.Datastores/redisCaches` |
| SQL Server dependency | Identified by port 1433/tcp transport or `sql`/`mssql`/`sqlserver` in image/name | Map to `Applications.Datastores/sqlDatabases` |
| Multiple Aspire projects | More than one AppHost `infra/` directory detected | Fail with FR-011 error message |
| Port binding | `ingress.targetPort` or `{{ targetPortOrDefault N }}` must be present | Use placeholder port, log as gap (FR-007) |
| Image reference | `template.containers[0].image` (typically `{{ .Image }}`) | Use `IMAGE_PLACEHOLDER` as default, log as assumption (FR-006) |
| Go template syntax | `{{ ... }}` expressions in YAML values | Strip/replace before YAML parsing (R-008) |

## Dependency Type Mapping Table

Dependencies are classified by heuristics applied to the `.tmpl.yaml` content (port number, transport protocol, container name/image).

| Heuristic | Radius Resource Type | Notes |
|---|---|---|
| Port 6379 + transport `tcp` (Redis container) | `Applications.Datastores/redisCaches` | Recipe-backed. Reference: `cache.tmpl.yaml` |
| `redis` in container image or resource name | `Applications.Datastores/redisCaches` | Recipe-backed |
| Port 1433 + transport `tcp` (SQL Server container) | `Applications.Datastores/sqlDatabases` | Recipe-backed. Reference: `sqlserver.tmpl.yaml` |
| `sql` or `mssql` or `sqlserver` in container image or resource name | `Applications.Datastores/sqlDatabases` | Recipe-backed |
| Port 5432 + transport `tcp` (PostgreSQL container) | Placeholder | No Portable Resource equivalent |
| `postgres` in container image or resource name | Placeholder | No Portable Resource equivalent |
| `mongo` in container image or resource name | `Applications.Datastores/mongoDatabases` | Recipe-backed (future) |
| `rabbitmq` in container image or resource name | `Applications.Messaging/rabbitMQQueues` | Recipe-backed (future) |
| Other / unrecognized | Placeholder with comment | Documented gap per FR-004 |
