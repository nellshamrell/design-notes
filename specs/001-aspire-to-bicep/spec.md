# Feature Specification: Aspire Manifest to Bicep Conversion

**Feature Branch**: `001-aspire-to-bicep`
**Created**: 2026-02-19
**Status**: Draft
**Input**: User description: "Radius CLI command that allows me to take an Aspire manifest file (use aspire-manifest.json as an example) and convert it to an app.bicep file that can be deployed with `rad deploy`"

## Clarifications

### Session 2026-02-19

- Q: Where should the command live in the `rad` CLI tree? → A: `rad aspire convert` — new top-level `aspire` command group.
- Q: Which Aspire resource types should be supported beyond containers? → A: Containers + known backing-service types (e.g., Redis, PostgreSQL) mapped to Radius data resources. Parameters/secrets and all other types emit warnings.
- Q: How should Aspire `external: true` bindings be handled? → A: Generate an `Applications.Core/gateways` resource for containers with external bindings.
- Q: What is explicitly out of scope for v1? → A: Cloud resource provisioning (Azure/AWS), Dapr integration configuration, service discovery, and parameter/secret handling are all out of scope.
- Q: How should Aspire resource entries with an `error` field (and no `type` field) be handled? → A: Skip the errored resource gracefully, emit a warning indicating the resource could not be generated in the manifest, and continue converting all remaining resources. The conversion must still succeed.
- Q: How should Aspire resources with `build.buildOnly: true` be handled? → A: Exclude them entirely from conversion. These are build-time-only artifacts (e.g., a frontend build step that produces static files consumed by another container via `containerFiles`). They do not represent runtime containers and should not produce a Radius resource. A warning MUST be emitted explaining the resource was skipped because it is build-only.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Convert a Basic Aspire Manifest (Priority: P1)

As a developer with an existing .NET Aspire application, I want to run a single CLI command that reads my Aspire manifest JSON file and produces a valid Radius `app.bicep` file so that I can deploy my application to Radius without manually authoring Bicep.

**Why this priority**: This is the core value proposition. Without basic conversion working end-to-end, no other stories deliver value. It enables the primary migration path from Aspire to Radius.

**Independent Test**: Can be fully tested by running the CLI command against a sample Aspire manifest and verifying the output Bicep file contains the expected application, container, and connection resources. The generated file can then be deployed with `rad deploy`.

**Acceptance Scenarios**:

1. **Given** a valid Aspire manifest JSON file containing container resources with bindings and environment variables, **When** the user runs the conversion command pointing to that file, **Then** a valid `app.bicep` file is produced that defines a Radius application with corresponding container resources, ports, and environment variable mappings.
2. **Given** a valid Aspire manifest, **When** the user runs `rad deploy` against the generated `app.bicep`, **Then** the deployment succeeds without Bicep compilation errors.
3. **Given** a valid Aspire manifest with inter-resource references (e.g., connection strings referencing other resources), **When** the conversion command runs, **Then** the generated Bicep file uses Radius `connections` to express those dependencies.
4. **Given** an Aspire manifest containing a `container.v1` resource with `build.buildOnly: true`, **When** the conversion command runs, **Then** that resource is excluded from the generated Bicep output entirely, a warning is emitted identifying it as a build-only artifact, and all other resources are converted normally.

---

### User Story 3 - Specify Output Path and Overwrite Behavior (Priority: P3)

As a developer, I want to control where the generated Bicep file is written and whether existing files are overwritten, so that I don't accidentally lose work.

**Why this priority**: Usability and safety are important for developer trust, but the feature is still useful with default output behavior alone.

**Independent Test**: Can be tested by running the command with an explicit output path flag and verifying the file appears at the specified location, and by running the command when a file already exists and verifying the appropriate prompt or error behavior.

**Acceptance Scenarios**:

1. **Given** no output path is specified, **When** the conversion command runs, **Then** the output file is written to `app.bicep` in the current working directory.
2. **Given** an explicit output path is provided via a flag, **When** the conversion command runs, **Then** the output file is written to the specified path.
3. **Given** the target output file already exists, **When** the conversion command runs without a force flag, **Then** the user is warned and the existing file is not overwritten.
4. **Given** the target output file already exists and a force/overwrite flag is provided, **When** the conversion command runs, **Then** the existing file is replaced.

---

### User Story 4 - Report Unsupported Aspire Resource Types (Priority: P3)

As a developer whose Aspire manifest contains resource types that have no direct Radius equivalent, I want clear warnings during conversion so I know what requires manual attention.

**Why this priority**: Transparency about conversion limitations prevents the user from deploying incomplete or incorrect Bicep. This improves trust and reduces debugging time.

**Independent Test**: Can be tested by adding an unsupported resource type to a manifest and running the conversion, verifying warning messages appear and the rest of the conversion still succeeds.

**Acceptance Scenarios**:

1. **Given** an Aspire manifest containing a resource type that cannot be mapped to a Radius resource, **When** the conversion command runs, **Then** the command completes successfully, includes a comment in the generated Bicep marking the unsupported resource, and prints a warning to the console listing the skipped resources.
2. **Given** an Aspire manifest where all resources are supported, **When** the conversion command runs, **Then** no warnings are printed.

---

### Edge Cases

- What happens when the input file does not exist or is not valid JSON? The command should exit with a clear error message and a non-zero exit code.
- What happens when the Aspire manifest JSON has no `resources` key or the resources object is empty? The command should produce a minimal valid Bicep file (application resource only) and warn that no resources were found.
- What happens when a container resource references another resource that is not defined in the manifest (dangling reference)? The command should warn about the unresolved reference and skip that connection.
- What happens when the manifest contains resource types from a newer Aspire schema version than the tool supports? The command should warn about the unrecognized schema version and attempt best-effort conversion.
- What happens when two Aspire resources would produce Radius resources with the same name? The command should detect the collision and disambiguate (e.g., by appending a suffix) or error clearly.
- What happens when the manifest contains a resource entry with an `error` field instead of a `type` field (e.g., `"docker-hub": { "error": "This resource does not support generation in the manifest." }`)? The command should skip that resource, emit a warning including the resource name and the error message, include a comment in the generated Bicep, and continue converting all other resources successfully.
- What happens when a `container.v1` resource has `build.buildOnly: true`? The command should exclude the resource entirely from conversion, emit a warning that the resource is a build-only artifact and was skipped, and include a comment in the generated Bicep. Build-only containers are not runtime containers — they produce artifacts (e.g., static files) consumed by other containers during build time.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The CLI MUST provide the command `rad aspire convert` (a new top-level `aspire` command group) that accepts an Aspire manifest JSON file path as input and produces a Radius-compatible Bicep file as output.
- **FR-002**: The command MUST parse Aspire manifest resources of type `container.v0` and `container.v1` and map them to `Applications.Core/containers` (or equivalent `Radius.Compute/containers`) Bicep resources with image, ports, environment variables, and command/args. The command MUST also parse known backing-service resource types (e.g., `redis.server.v0`, `postgres.server.v0`) and map them to the corresponding Radius data-store resources (e.g., `Radius.Data/redisCaches`, `Radius.Data/postgreSqlDatabases`). All other unrecognized resource types MUST emit warnings per FR-012.
- **FR-003**: The command MUST map Aspire container `bindings` (with scheme, protocol, and targetPort) to Radius container `ports` definitions. Bindings marked `external: true` MUST additionally trigger gateway resource generation per FR-016.
- **FR-004**: The command MUST resolve Aspire expression references (e.g., `{cache.bindings.tcp.host}`) in environment variables and convert them to Bicep resource references in the output.
- **FR-005**: The command MUST generate a top-level Radius application resource and wire all container resources to it.
- **FR-007**: The command MUST generate the required Bicep extension declarations (e.g., `extension radius`) at the top of the output file.
- **FR-008**: The command MUST generate a Bicep `environment` parameter so the output is compatible with `rad deploy`.
- **FR-009**: The command MUST accept an optional output path flag; if not provided, it defaults to `app.bicep` in the current directory.
- **FR-010**: The command MUST refuse to overwrite an existing file unless a force/overwrite flag is provided.
- **FR-011**: The command MUST print a summary after conversion listing the number of resources converted and any warnings.
- **FR-012**: The command MUST exit with a non-zero exit code and a descriptive error message when the input file is missing, unreadable, or not valid JSON.
- **FR-013**: The command MUST warn (to stderr or console) when it encounters an Aspire resource type it cannot map, and include a comment in the generated Bicep at the location where that resource would appear.
- **FR-014**: The command MUST map `container.v1` resources with `build` configurations (where `build.buildOnly` is absent or `false`) to Radius container resources, using the image reference pattern appropriate for pre-built images (since Radius does not build images). A warning MUST be emitted advising the user to build and push the image separately.
- **FR-019**: The command MUST exclude `container.v1` resources whose `build` configuration contains `"buildOnly": true`. These resources are build-time-only artifacts that do not represent runtime containers. The command MUST skip the resource entirely (no Radius resource generated), emit a warning to the console identifying the resource as build-only, and include a comment in the generated Bicep noting the exclusion. The `buildOnly` check MUST take precedence over the normal container mapping in FR-014.
- **FR-015**: The command MUST map inter-resource connection strings (e.g., `{cache.connectionString}`) to Radius `connections` on the consuming container resource.
- **FR-016**: The command MUST maintain an explicit mapping table of supported Aspire backing-service resource types to Radius resource types. Initially this MUST include at minimum: Redis → `Radius.Data/redisCaches`, PostgreSQL → `Radius.Data/postgreSqlDatabases`, MySQL → `Radius.Data/mySqlDatabases`. The mapping table MUST be extensible for future additions.
- **FR-017**: The command MUST generate an `Applications.Core/gateways` (or equivalent `Radius.Compute/routes`) resource for any container whose Aspire bindings include `external: true`. The gateway MUST route to the container’s corresponding port. If multiple bindings on the same container are external, a single gateway with multiple routes MUST be generated.- **FR-018**: The command MUST gracefully handle Aspire manifest resource entries that contain an `error` field instead of a `type` field. Such resources MUST be skipped, a warning MUST be emitted to the console including the resource name and the error message text, and a comment MUST be included in the generated Bicep file noting the skipped resource. The presence of errored resources MUST NOT prevent the conversion of remaining valid resources.

**Resource Exclusion Priority**: When evaluating a resource, the command MUST apply exclusion checks in the following order: (1) error field present → skip per FR-018, (2) `build.buildOnly: true` → skip per FR-019, (3) unsupported type → warn per FR-013, (4) supported type → map normally.
### Key Entities

- **Aspire Manifest**: A JSON document conforming to the Aspire manifest schema. Contains a `resources` map where each entry typically has a `type` (e.g., `container.v0`, `container.v1`, `redis.server.v0`, `postgres.server.v0`), optional `bindings`, `env`, `connectionString`, `image`, and other properties depending on the type. Some resource entries may instead contain an `error` field (with no `type`) indicating the Aspire manifest publisher could not generate that resource (e.g., custom Docker registries or unsupported integrations). Container resources with `build.buildOnly: true` are build-time-only artifacts that produce files consumed by other containers but do not run as independent services.
- **Radius Bicep File**: A `.bicep` file using Radius extensions that defines an application, its container resources, connections, and supporting resources. Deployable via `rad deploy`.
- **Resource Mapping**: The association between an Aspire resource type/configuration and its corresponding Radius Bicep resource definition. Each mapping transforms Aspire-specific properties into Radius-specific properties.
- **Expression Reference**: An Aspire manifest interpolation pattern (e.g., `{resource.bindings.port.host}`) that must be resolved to a Bicep reference expression in the output.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can convert the provided sample `aspire-manifest.json` to a valid `app.bicep` and deploy it with `rad deploy` in under 2 minutes total (conversion + deploy initiation).
- **SC-002**: The generated Bicep file compiles without errors when processed by the Bicep toolchain used by Radius.
- **SC-003**: 100% of supported Aspire resource types in the sample manifest produce correctly mapped Radius resources in the output.
- **SC-004**: Unsupported resource types produce clear, actionable warning messages that identify the resource name and type, enabling the user to address them manually.
- **SC-005**: The command provides a conversion summary (resources converted, warnings) so the user can verify completeness at a glance.

## Assumptions

- The user has already generated the Aspire manifest JSON (e.g., via `dotnet run --publisher manifest`) before running this command. The CLI does not invoke Aspire tooling directly.
- Container images referenced in `container.v1` resources with `build` configurations are assumed to already be built and pushed to an accessible registry. The conversion tool does not build images.
- The conversion targets the current Radius Bicep resource schema (e.g., `Applications.Core/containers@2023-10-01-preview` or `Radius.Compute/containers@2025-08-01-preview`). The specific API version used will follow the version conventions active at the time of implementation.
- The `rad deploy` command and Radius environment are already set up and functional. This feature only covers the file conversion step.
- The Aspire manifest conforms to a known schema version (e.g., `aspire-8.0.json`). Graceful degradation is expected for unknown schema versions.
- Some Aspire manifest resource entries may contain an `error` field instead of a `type` field. This occurs when the Aspire manifest publisher cannot serialize a resource (e.g., custom Docker registries, unsupported integrations). The conversion tool treats these as skippable entries.
- `annotated.string` resource types (e.g., URI encoding filters) are treated as pass-through values — the filter behavior is noted in comments but not replicated in the Bicep output.
- Aspire `container.v1` resources with `build.buildOnly: true` are build-time artifacts (e.g., a frontend build step that produces static files injected into another container via `containerFiles`). These are not runtime containers and should not be deployed to Radius.

## Out of Scope (v1)

The following capabilities are explicitly excluded from the initial version of this feature. Aspire manifest resources or configurations related to these areas MUST trigger unsupported-resource warnings (per FR-012) rather than silent omission.

- **Cloud resource provisioning**: Azure, AWS, or GCP resource definitions in Aspire manifests (e.g., `azure.bicep.v0`, `azure.storage.v0`, `aws.sqs.v0`) are not converted. Users must provision these separately and wire them into the Radius environment.
- **Dapr integration configuration**: Aspire Dapr component resources (e.g., `dapr.component.v0`) are not mapped. Dapr sidecar configuration in Radius is handled through Radius extensions and is outside this conversion tool's scope.
- **Service discovery**: Aspire's automatic service discovery and URL resolution between resources is not replicated. The generated Bicep uses explicit Radius `connections` for inter-resource communication instead.
- **Image building**: The tool does not build container images. `container.v1` resources with `build` configurations produce a Radius container resource referencing a placeholder image with a warning (per FR-013).
- **Aspire toolchain invocation**: The tool does not run `dotnet` commands or invoke the Aspire manifest publisher. The user must provide a pre-generated manifest JSON file.
- **Parameter and secret handling**: Aspire `parameter.v0` resources (including those with `secret: true` inputs) are not mapped to Bicep `@secure()` parameters. Parameter resources in the manifest will be treated as unsupported resource types and emit warnings. Users must manually add any required Bicep parameters for secrets after conversion.
