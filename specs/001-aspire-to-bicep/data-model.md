# Data Model: Aspire Manifest to Bicep Conversion

**Feature**: 001-aspire-to-bicep
**Date**: 2026-02-19

## Overview

This document defines the internal data structures used during the conversion pipeline: **Parse → Map → Emit**. These are Go structs internal to the conversion tool, not persisted or exposed via API.

## Entity: AspireManifest

The top-level parsed representation of an Aspire manifest JSON file.

| Field | Type | Description |
|-------|------|-------------|
| Schema | string | The `$schema` URL from the manifest (e.g., `aspire-8.0.json`). Used for version detection. |
| Resources | map[string]AspireResource | Map of resource name → resource definition. Key is the logical resource name (e.g., `cache`, `app`). |

## Entity: AspireResource

A single resource entry within the Aspire manifest.

| Field | Type | Description |
|-------|------|-------------|
| Name | string | The resource's logical name (from the map key, not a field in JSON). |
| Type | string | Resource type identifier (e.g., `container.v0`, `parameter.v0`, `redis.server.v0`). |
| Image | string | Container image reference (for container types). Empty for non-containers. |
| Entrypoint | string | Container entrypoint override. Optional. |
| Args | []string | Container command arguments. Optional. |
| Env | map[string]string | Environment variables. Values may contain expression references like `{cache.bindings.tcp.host}`. |
| Bindings | map[string]AspireBinding | Named network bindings (ports/endpoints). |
| ConnectionString | string | Connection string template. May contain expression references. |
| Build | *AspireBuild | Build configuration (for `container.v1` with Dockerfile). Nil if not present. |
| Value | string | Parameter value (for `parameter.v0`). May contain expression references. |
| Inputs | map[string]AspireInput | Parameter inputs (for `parameter.v0`). |
| Error | string | Error message from the Aspire manifest publisher (present when the resource could not be generated). When non-empty, the resource has no `type` and should be skipped during conversion. |

## Entity: AspireBinding

A network binding/endpoint on a container resource.

| Field | Type | Description |
|-------|------|-------------|
| Name | string | Binding name (from map key, e.g., `tcp`, `http`). |
| Scheme | string | Protocol scheme (e.g., `tcp`, `http`, `https`). |
| Protocol | string | Network protocol (e.g., `tcp`). |
| Transport | string | Transport protocol (e.g., `tcp`, `http`). |
| TargetPort | int | Container port number. |
| External | bool | Whether this binding is externally accessible. |

## Entity: AspireBuild

Build configuration for `container.v1` resources.

| Field | Type | Description |
|-------|------|-------------|
| Context | string | Build context directory path. |
| Dockerfile | string | Dockerfile path relative to context. |
| BuildOnly | bool | If true, the container is a build artifact only (no runtime). Resources with `BuildOnly: true` MUST be excluded from conversion entirely — they do not produce a Radius resource. |

## Entity: AspireInput

A parameter input definition. Present in the parse model for manifest completeness, but `parameter.v0` resource mapping is out of scope for v1 (these resources are treated as unsupported).

| Field | Type | Description |
|-------|------|-------------|
| Type | string | Input type (e.g., `string`). |
| Secret | bool | Whether this input is a secret value. |
| Default | *AspireInputDefault | Default value configuration. Nil if no default. |

## Entity: AspireInputDefault

Default value generation configuration for parameter inputs.

| Field | Type | Description |
|-------|------|-------------|
| Generate | *AspireGenerate | Auto-generation configuration. Nil if value is static. |

## Entity: AspireGenerate

Auto-generation configuration for default values.

| Field | Type | Description |
|-------|------|-------------|
| MinLength | int | Minimum length for generated value. |
| Special | bool | Whether to include special characters. |

---

## Intermediate Representation (Bicep IR)

These entities represent the conversion output before text emission.

## Entity: BicepFile

The complete Bicep file to be emitted.

| Field | Type | Description |
|-------|------|-------------|
| Extensions | []string | Required Bicep extension names (e.g., `radius`, `containers`). |
| Parameters | []BicepParameter | Declared Bicep parameters (environment, secure params). |
| Application | BicepResource | The Radius application resource. |
| Containers | []BicepContainer | Container resources. |
| DataStores | []BicepResource | Data-store resources (Redis, PostgreSQL, MySQL). |
| Gateways | []BicepGateway | Gateway/route resources for external bindings. |
| Comments | []BicepComment | Comments for unsupported/skipped resources. |
| Warnings | []string | Warning messages to display to the user (not in the Bicep file). |

## Entity: BicepParameter

A Bicep parameter declaration.

| Field | Type | Description |
|-------|------|-------------|
| Name | string | Parameter name (e.g., `environment`, `cachePassword`). |
| Type | string | Bicep type (e.g., `string`). |
| Secure | bool | Whether to emit `@secure()` decorator. Reserved for future use (parameter.v0 mapping is out of scope for v1). |
| Description | string | Parameter description for `@description()` decorator. |

## Entity: BicepResource

A generic Bicep resource declaration.

| Field | Type | Description |
|-------|------|-------------|
| SymbolicName | string | Bicep symbolic name (identifier used in Bicep code). |
| TypeName | string | Full resource type (e.g., `Radius.Core/applications@2025-08-01-preview`). |
| Name | string | Resource name in Radius (the `name` property). |
| Properties | map[string]any | Resource properties (environment, application ref, etc.). |

## Entity: BicepContainer

A Radius container resource with full container-specific properties.

| Field | Type | Description |
|-------|------|-------------|
| SymbolicName | string | Bicep symbolic name. |
| TypeName | string | Full resource type. |
| Name | string | Container name. |
| Image | string | Container image reference. |
| Ports | map[string]BicepPort | Port definitions mapped from Aspire bindings. |
| Env | map[string]BicepEnvVar | Environment variables (resolved from Aspire expressions). |
| Command | []string | Entrypoint + args if specified. |
| Connections | map[string]BicepConnection | Radius connections to other resources. |
| ApplicationRef | string | Bicep reference to the application resource. |
| EnvironmentRef | string | Bicep reference to the environment parameter. |
| NeedsBuildWarning | bool | Whether to emit a "build image separately" warning comment. |
| BuildContext | string | Original Aspire build context path (for the warning comment). |

## Entity: BicepPort

A container port definition.

| Field | Type | Description |
|-------|------|-------------|
| ContainerPort | int | The target port number. |
| Protocol | string | Protocol (e.g., `TCP`). |

## Entity: BicepEnvVar

An environment variable in a container.

| Field | Type | Description |
|-------|------|-------------|
| Value | string | Static value (if no reference). |
| BicepExpression | string | A Bicep expression (if resolved from an Aspire reference). Only one of Value or BicepExpression is set. |

## Entity: BicepConnection

A Radius connection to another resource.

| Field | Type | Description |
|-------|------|-------------|
| Source | string | Bicep reference to the target resource (e.g., `cache.id`). |

## Entity: BicepGateway

A gateway/route resource for external bindings.

| Field | Type | Description |
|-------|------|-------------|
| SymbolicName | string | Bicep symbolic name. |
| TypeName | string | Full resource type (e.g., `Radius.Compute/routes@2025-08-01-preview`). |
| Name | string | Gateway name. |
| ContainerRef | string | Bicep reference to the container resource. |
| Routes | []BicepGatewayRoute | Route definitions. |
| ApplicationRef | string | Bicep reference to the application. |
| EnvironmentRef | string | Bicep reference to the environment. |

## Entity: BicepGatewayRoute

A single route within a gateway.

| Field | Type | Description |
|-------|------|-------------|
| Path | string | URL path (default `/`). |
| Port | int | Target port on the container. |

## Entity: BicepComment

A comment block in the generated Bicep for skipped/unsupported resources.

| Field | Type | Description |
|-------|------|-------------|
| ResourceName | string | The Aspire resource name that was skipped. |
| ResourceType | string | The Aspire resource type that was unsupported. |
| Message | string | Human-readable explanation. |

---

## Relationships

```
AspireManifest 1──* AspireResource
AspireResource 1──* AspireBinding
AspireResource 1──* AspireInput
AspireResource 0..1── AspireBuild
AspireInput 0..1── AspireInputDefault
AspireInputDefault 0..1── AspireGenerate

BicepFile 1──* BicepParameter
BicepFile 1──1 BicepResource (Application)
BicepFile 1──* BicepContainer
BicepFile 1──* BicepResource (DataStores)
BicepFile 1──* BicepGateway
BicepFile 1──* BicepComment

BicepContainer *──* BicepConnection (many connections per container)
BicepContainer 1──* BicepPort
BicepContainer 1──* BicepEnvVar
BicepGateway 1──* BicepGatewayRoute
```

## State Transitions

The conversion is a single-pass stateless pipeline with no persistent state:

```
Input JSON file → Parse → AspireManifest
    → Map → BicepFile (IR)
    → Emit → .bicep text string
    → Write → Output file
```

No state machines, no lifecycle management, no persistence.

## Validation Rules

| Entity | Rule |
|--------|------|
| AspireManifest | Must have non-nil `Resources` map |
| AspireResource | `Type` field must be non-empty |
| AspireResource (container) | `Image` must be non-empty for `container.v0`; may be empty for `container.v1` with `Build` |
| AspireResource (container) | If `Build.BuildOnly` is `true`, the resource MUST be skipped during mapping (not converted to a Radius resource) |
| AspireBinding | `TargetPort` must be > 0 |
| BicepParameter | `Name` must be a valid Bicep identifier (alphanumeric + underscore, starts with letter) |
| BicepContainer | `SymbolicName` must be unique across all resources in the BicepFile |
| BicepFile | Must have exactly one Application resource |
| BicepFile | `Extensions` must include at least `radius` |
