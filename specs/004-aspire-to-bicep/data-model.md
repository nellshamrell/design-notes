# Data Model: Aspire-to-Bicep PoC

**Branch**: `004-aspire-to-bicep` | **Date**: 2026-02-23

## Entity Overview

The conversion pipeline has two model layers: the **Parsed Model** (representation of input Bicep files) and the **Radius Model** (representation of the target app.bicep output). The mapper transforms one into the other, and the reporter tracks the lineage between them.

## Parsed Model (Input)

These entities represent the structure extracted from `azd infra synth` Bicep files.

### BicepFile

Represents a single parsed Bicep file.

| Field | Type | Description |
|-------|------|-------------|
| `Path` | `string` | Absolute file path of the source Bicep file |
| `Resources` | `[]BicepResource` | Resource declarations found in the file |
| `Parameters` | `[]BicepParameter` | Parameter declarations found in the file |
| `Modules` | `[]BicepModule` | Module declarations found in the file (main.bicep only) |
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
| `Name` | `string` | Application name derived from Aspire project name | `main.bicep` → `environmentName` param or directory name |
| `Containers` | `[]RadiusContainer` | Service containers in the application | Per-service Bicep modules |
| `Dependencies` | `[]RadiusDependency` | Infrastructure dependencies (Redis, etc.) | Per-dependency Bicep modules |
| `Parameters` | `[]RadiusParameter` | Bicep parameters for the output file | Derived from image refs, secrets |
| `Variables` | `[]RadiusVariable` | Bicep variables for the output file | Derived from computed values |

### RadiusContainer

Represents an `Radius.Compute/containers` resource.

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Container resource name | Container App `name` |
| `ImageParam` | `string` | Bicep parameter name for the image | Derived: `{name}Image` |
| `ImageDefault` | `string` | Default image value | Container App `template.containers[0].image` or `{name}:latest` |
| `Ports` | `[]RadiusPort` | Port definitions | Container App `configuration.ingress` |
| `EnvVars` | `map[string]string` | Environment variables | Container App `template.containers[0].env[]` |
| `Connections` | `[]RadiusConnection` | Connections to other resources | Derived from `ConnectionStrings__*` env vars |
| `IsExternal` | `bool` | Whether this service has external ingress | Container App `ingress.external` |
| `Command` | `[]string` | Container command override | Container App `template.containers[0].command` |

### RadiusPort

Represents a port definition on a container.

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Port name (e.g., `http`, `tcp`) | Derived from transport type |
| `ContainerPort` | `int` | Port number | `ingress.targetPort` |
| `Protocol` | `string` | Protocol (`TCP`, `UDP`) | `ingress.transport` |
| `IsPlaceholder` | `bool` | Whether this port is a placeholder (not found in source) | Gap detection |

### RadiusConnection

Represents a connection from one resource to another.

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Connection name (e.g., `cache`, `api`) | Derived from `ConnectionStrings__` suffix |
| `TargetResourceName` | `string` | Symbolic name of the target resource | Parsed from connection string or env var |
| `Source` | `string` | Bicep expression for the connection source (e.g., `cache.id`) | Mapper output |

### RadiusDependency

Represents a portable resource (e.g., Redis).

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `Name` | `string` | Resource name | Redis container/resource name |
| `Type` | `string` | Radius resource type (e.g., `Applications.Datastores/redisCaches`) | Mapped from Azure resource type |
| `IsRecipeBacked` | `bool` | Whether provisioned by Recipe | Always `true` for PoC |
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
azd infra synth Bicep files
    → [Parser] → []BicepFile (Parsed Model)
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
| Input directory | Must exist and contain at least one `.bicep` file | Fail with FR-010 error message |
| Input directory | Must contain a `main.bicep` | Fail with FR-010 error message |
| BicepResource | Must have a valid `Type` field | Skip resource, log as gap |
| Container App | Must have `template.containers` array | Log port/image as gaps, generate placeholder |
| Redis resource | Must be `Microsoft.Cache/redis` or Redis container image | Map to `Applications.Datastores/redisCaches` |
| Multiple Aspire projects | More than one `main.bicep` or multiple managed environments | Fail with FR-011 error message |
| Port binding | `ingress.targetPort` must be present | Use placeholder port, log as gap (FR-007) |
| Image reference | `template.containers[0].image` | Use `{name}:latest` default, log as assumption (FR-006) |

## Dependency Type Mapping Table

| Azure Resource Type | Radius Resource Type | Notes |
|---|---|---|
| `Microsoft.Cache/redis` | `Applications.Datastores/redisCaches` | Recipe-backed |
| Redis container image (`redis:*`) | `Applications.Datastores/redisCaches` | Recipe-backed |
| `Microsoft.DocumentDB/databaseAccounts` (MongoDB) | `Applications.Datastores/mongoDatabases` | Recipe-backed (future) |
| `Microsoft.Sql/servers` | `Applications.Datastores/sqlDatabases` | Recipe-backed (future) |
| `Microsoft.DBforPostgreSQL/flexibleServers` | Placeholder | No Portable Resource equivalent |
| Other Azure resources | Placeholder with comment | Documented gap per FR-004 |
