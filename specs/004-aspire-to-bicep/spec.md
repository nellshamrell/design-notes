# Feature Specification: Aspire-to-Bicep PoC CLI Command

**Feature Branch**: `004-aspire-to-bicep`
**Created**: 2026-02-23
**Status**: Draft
**Input**: User description: "Build a proof-of-concept of a rad cli command that takes a simple Aspire application and produces a Radius-deployable application definition file (app.bicep) by using Aspire's generated infrastructure artifacts as the starting point."

## Clarifications

### Session 2026-02-23

- Q: Which Aspire artifact format should the conversion use as its primary input — Aspire Manifest JSON, azd-generated Bicep, or both? → A: azd-generated Bicep files (`azd infra synth` output).
- Q: How should the conversion model Redis (and similar dependencies) in the output app.bicep — Portable Resource, manual placeholder, or Portable Resource with fallback? → A: Portable Resource (`Applications.Datastores/redisCaches`), Recipe-backed.
- Q: Should the PoC conversion be implemented as a `rad` CLI subcommand or as a standalone tool? → A: `rad` CLI subcommand (e.g., `rad bicep generate --from-aspire`).
- Q: How should the conversion handle container image references in the output app.bicep? → A: Bicep parameters with `{project-name}:latest` as default value, overridable at deploy time.
- Q: What format should the mapping/gap report be produced in — console output, companion Markdown file, or both? → A: Both console output and companion Markdown file.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Happy-Path Conversion and Deployment (Priority: P1)

As a developer with an existing simple Aspire solution (the reference application at `./example-aspire-app` containing a frontend, API, Redis, and SQL Server), I want to run a single conversion command that reads the Aspire-generated infrastructure artifacts and produces a Radius-compatible app.bicep file, so that I can deploy the same logical application using Radius without manually authoring Bicep from scratch.

The reference application (`./example-aspire-app`) has this topology defined in `AspireApp.AppHost/AppHost.cs`:
- `cache` — Redis (`AddRedis`)
- `sqlserver` — SQL Server with `weatherdb` database (`AddSqlServer` + `AddDatabase`)
- `apiservice` — .NET project, references `sqlserver`, waits for `sqlserver`, health check at `/health`
- `webfrontend` — .NET project (external endpoints), references `apiservice` + `cache`, waits for both, health check at `/health`

**Why this priority**: This is the core value proposition of the PoC. Without a working end-to-end conversion and deployment, no other story delivers meaningful value. It validates feasibility and produces the primary deliverable.

**Independent Test**: Can be fully tested by running the conversion command on the reference application directory (`./example-aspire-app`), then deploying the resulting app.bicep to a local Radius environment. Delivers a deployed Radius application with working webfrontend → apiservice, webfrontend → cache, and apiservice → sqlserver connectivity.

**Acceptance Scenarios**:

1. **Given** the reference Aspire application at `./example-aspire-app` with a webfrontend service, an apiservice, a Redis dependency (`cache`), and a SQL Server dependency (`sqlserver` with `weatherdb` database), **When** the developer runs the conversion command pointing at the application directory, **Then** a single app.bicep file is produced that contains an Application resource, two Container resources (webfrontend, apiservice), a Redis Portable Resource (`cache`), a SQL Database Portable Resource (`sqlserver`), and Connection definitions reflecting webfrontend→apiservice, webfrontend→cache, and apiservice→sqlserver relationships.
2. **Given** the generated app.bicep file, **When** the developer deploys it using `rad deploy`, **Then** the Radius application is created successfully, the webfrontend endpoint is reachable, the webfrontend can call the apiservice, and the cache and sqlserver dependencies are Recipe-provisioned.
3. **Given** a successful deployment, **When** the developer inspects the Radius application graph (`rad app show`), **Then** the displayed topology matches the original Aspire application's logical structure (two services, two dependencies, correct connections).

---

### User Story 2 - Handling Missing or Ambiguous Aspire Artifacts (Priority: P2)

As a developer running the conversion on an Aspire solution where some expected information is missing or ambiguous (e.g., service ports not specified, dependency credentials not extractable, image tags unclear), I want the conversion to clearly report what data it could not resolve and produce a safe fallback with documented placeholders, so that I am never silently given an incorrect app.bicep.

**Why this priority**: Robustness and transparency are essential for trust. A tool that silently guesses wrong values is worse than one that fails loudly. This story ensures the PoC is honest about its limitations.

**Independent Test**: Can be tested by modifying or stripping specific fields from the Aspire artifacts (e.g., remove port bindings from a service definition) and running the conversion. The output app.bicep should contain clearly marked placeholder values (e.g., `/* PLACEHOLDER: port not found in Aspire artifacts — set manually */`) and the command's console output should list every gap detected.

**Acceptance Scenarios**:

1. **Given** an Aspire solution where service port/binding information is missing from the artifacts, **When** the developer runs the conversion command, **Then** the output app.bicep uses a documented placeholder for the port value and the console output lists the missing information with guidance on how to resolve it.
2. **Given** an Aspire solution where dependency connection details (host, port, credentials) are not directly extractable, **When** the developer runs the conversion command, **Then** the app.bicep models the dependency with placeholder connection properties and the console output clearly identifies each missing connection detail.
3. **Given** an Aspire solution where container image names differ between local build and deployment contexts, **When** the developer runs the conversion command, **Then** the app.bicep uses a configurable image reference (e.g., a Bicep parameter with a sensible default) and the console output documents the assumption made.

---

### User Story 3 - Repeatable Conversion with Consistent Output (Priority: P3)

As a platform engineer evaluating standardization of Aspire-to-Radius conversion, I want the conversion to produce identical logical topology in the app.bicep every time I regenerate the Aspire solution from the same source and re-run the command, so that I can trust the process is deterministic and suitable for automation.

**Why this priority**: Repeatability is a prerequisite for any standardized pipeline. Without it, the PoC cannot be promoted beyond a one-off demo. This story validates automation readiness.

**Independent Test**: Can be tested by generating the Aspire solution from identical source, running the conversion, capturing the output, then repeating the entire process from scratch and diff-ing the two app.bicep files. The logical topology (resources, connections, naming) must match.

**Acceptance Scenarios**:

1. **Given** two independently generated Aspire solutions from the same source definition, **When** the conversion command is run on each, **Then** both resulting app.bicep files contain the same set of resources, the same connections, and the same logical structure (cosmetic differences like timestamps or ordering are acceptable).
2. **Given** a conversion run that produced an app.bicep, **When** the same conversion is re-run without any changes to the Aspire artifacts, **Then** the output app.bicep is byte-for-byte identical to the previous output.

---

### User Story 4 - Artifact-to-Resource Mapping Documentation (Priority: P2)

As a developer or platform engineer new to Radius, I want the conversion process to produce or reference a clear mapping table that shows which Aspire artifact was used to populate each Radius resource property, so that I can understand the translation logic and manually adjust the output if needed.

**Why this priority**: The primary learning outcome of this PoC is the mapping table. Without it, future work cannot build on the PoC's findings. This is co-prioritized with Story 2 because documentation of the process is as valuable as the process itself.

**Independent Test**: Can be tested by running the conversion and verifying that a mapping report is produced (either as console output, a companion markdown file, or both) that maps each Radius resource property to its source Aspire artifact field.

**Acceptance Scenarios**:

1. **Given** a successful conversion run, **When** the developer examines the mapping output, **Then** every populated field in app.bicep has a corresponding entry identifying the Aspire artifact and field it was derived from.
2. **Given** a conversion with gaps, **When** the developer examines the mapping output, **Then** every placeholder or missing field has a corresponding entry explaining what Aspire artifact was expected but not found, and what manual input is needed.

---

### Edge Cases

- **Missing service port/binding information**: The Aspire artifacts may not specify explicit ports for a service (e.g., relying on Aspire runtime defaults). The conversion must detect this absence and insert a documented placeholder rather than guessing a port number.
- **Unresolvable dependency credentials**: The Aspire artifacts may reference a Redis or SQL Server dependency without exposing connection strings or credentials (these may be injected at runtime by Aspire). The conversion must flag these as gaps requiring manual input.
- **Image name/tag mismatch**: Local development may use `project-name:latest` or a project reference while deployment artifacts reference a registry path. The conversion emits Bicep parameters with `{project-name}:latest` as the default, overridable at deploy time. The mapping report documents this assumption.
- **Unsupported dependency type**: If the reference application included a dependency type with no direct Radius Portable Resource equivalent (e.g., a custom or uncommon database), the conversion must emit a clearly commented placeholder resource and a warning, rather than omitting the dependency silently. The current reference application (`./example-aspire-app`) uses Redis and SQL Server, both of which have Portable Resource equivalents.
- **Go template expressions in YAML**: The `.tmpl.yaml` files contain Go template syntax (`{{ .Image }}`, `{{ securedParameter "..." }}`, etc.) that must be handled during YAML parsing. The parser strips or replaces these expressions before parsing and extracts any embedded values (e.g., default port from `{{ targetPortOrDefault 8080 }}`).
- **Multiple connection string patterns**: The reference application uses both `ConnectionStrings__<name>` (for Redis and SQL Server) and `services__<name>__http__0` (for service-to-service) env var patterns. The conversion must recognize both patterns when deriving connections.
- **Empty or malformed Aspire artifacts directory**: The conversion command must fail with a clear error message if the input directory is empty, missing expected files, or contains unparseable content.
- **Multiple Aspire projects in one solution**: The PoC scopes to a single Aspire project. If multiple projects are detected, the command must fail with a clear message indicating this is out of scope.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The conversion MUST be implemented as a `rad` CLI subcommand (e.g., `rad bicep generate --from-aspire <path>`). It MUST accept a path to an Aspire application directory containing `azd infra synth` output artifacts (solution-level `infra/` directory with `main.bicep` and AppHost-level `<AppHost>/infra/` directory with per-service `.tmpl.yaml` templates) as input and produce a single app.bicep file as output.
- **FR-002**: The output app.bicep MUST contain a valid Radius `Applications.Core/applications` resource representing the application.
- **FR-003**: The output app.bicep MUST contain an `Applications.Core/containers` resource for each service defined in the Aspire application (webfrontend and apiservice for the reference application scope).
- **FR-004**: The output app.bicep MUST model recognized dependencies using Radius Portable Resources (e.g., Redis → `Applications.Datastores/redisCaches`, SQL Server → `Applications.Datastores/sqlDatabases`). The Portable Resource is Recipe-backed, meaning the Radius environment provides the actual infrastructure via a configured Recipe. For dependency types that have no Radius Portable Resource equivalent, the conversion MUST emit a clearly documented placeholder with explanation of why a native mapping is not possible.
- **FR-005**: The output app.bicep MUST include connection definitions that reflect the service-to-service (webfrontend→apiservice) and service-to-dependency (webfrontend→cache, apiservice→sqlserver) relationships present in the Aspire application. Connections are derived from `ConnectionStrings__*` and `services__*__http__*` environment variable patterns in the `.tmpl.yaml` templates.
- **FR-006**: The conversion MUST emit a Bicep `param` declaration for each service's container image, using `{project-name}:latest` as the default value (e.g., `param frontendImage string = 'frontend:latest'`). The Container resource MUST reference this parameter for its `container.image` property, allowing the developer to override the image at deploy time via `rad deploy --parameters`.
- **FR-007**: The conversion MUST extract port/binding information from the Aspire artifacts and map it to the `container.ports` property. When port information is missing, the command MUST insert a documented placeholder and emit a warning.
- **FR-008**: The conversion MUST produce a mapping report in two formats: (1) a summary printed to the console during the conversion run, and (2) a companion Markdown file written alongside the output app.bicep (e.g., `mapping-report.md`). The report MUST document, for every populated field in app.bicep, which Aspire artifact and field it was derived from.
- **FR-009**: The conversion MUST produce a gap report (included in both the console output and the companion Markdown file) listing every field that could not be populated from Aspire artifacts, including what was expected and what manual action is needed.
- **FR-010**: The conversion MUST fail with a clear, actionable error message when the input directory is empty, missing expected artifact files, or contains unparseable content.
- **FR-011**: The conversion MUST fail with a clear message when multiple Aspire projects are detected, indicating this is out of scope for the PoC.
- **FR-012**: The conversion MUST be idempotent — running the command twice on identical Aspire artifacts MUST produce identical app.bicep output.
- **FR-013**: The output app.bicep MUST be deployable using `rad deploy` in a local Radius environment without manual edits (assuming all placeholders have been resolved).

### Key Entities

- **Aspire Infrastructure Artifacts**: The set of files generated by running `azd infra synth` (Azure Developer CLI infrastructure synthesis) against an Aspire solution. These consist of: (1) solution-level Bicep files in `infra/` (`main.bicep`, `resources.bicep`, `main.parameters.json`) describing Azure infrastructure scaffolding, and (2) per-service YAML templates in `<AppHost>/infra/` (`.tmpl.yaml` files) defining Azure Container App resources with Go template syntax. The per-service YAML templates are the primary input for service-level conversion. Note: the Aspire Manifest JSON (`--publisher manifest`) is not used as input.
- **Radius Application (app.bicep)**: The output file describing the application in Radius's resource model. Contains an application resource, container resources for each service, dependency resources, and connection definitions. This is the primary output of the conversion.
- **Mapping Report**: A structured document produced in two formats: (1) a console summary printed during the conversion run, and (2) a companion Markdown file (e.g., `mapping-report.md`) written alongside app.bicep. The report records the lineage of every field in the output app.bicep — tracing it back to the specific Aspire artifact (`.tmpl.yaml` file, `main.bicep`, or `AppHost.cs`) and field it was derived from, or marking it as a gap requiring manual input. The companion file is suitable for version control, peer review, and documentation.
- **Service**: A compute workload in the application (e.g., webfrontend, apiservice). In Aspire, represented as a project resource with `AddProject<>()` in `AppHost.cs` and a corresponding `.tmpl.yaml` template. In Radius, represented as a `Radius.Compute/containers` resource.
- **Dependency**: An infrastructure component consumed by services (e.g., Redis, SQL Server). In Aspire, represented as a resource added with `AddRedis()`, `AddSqlServer()`, etc. in `AppHost.cs` and a corresponding `.tmpl.yaml` template. In Radius, modeled as a Portable Resource (e.g., `Applications.Datastores/redisCaches`, `Applications.Datastores/sqlDatabases`) that is provisioned by a Recipe configured in the Radius environment. For dependency types without a Portable Resource equivalent, a documented placeholder is used instead.
- **Connection**: A declared relationship between two resources (service→service or service→dependency). In Aspire, represented by `WithReference()` calls in `AppHost.cs` and manifested as `ConnectionStrings__*` or `services__*__http__*` environment variables in the `.tmpl.yaml` templates. In Radius, represented by the `connections` property on Container resources.

## Assumptions

- The Aspire solution is the reference application at `./example-aspire-app` (generated via `dotnet new aspire-starter` with additional SQL Server dependency) or a similarly structured application.
- Aspire infrastructure artifacts are generated using Azure Developer CLI (`azd infra synth`) and are available on the local filesystem. The output consists of solution-level Bicep files (`infra/`) and per-service YAML templates (`<AppHost>/infra/*.tmpl.yaml`). The Aspire Manifest JSON format is not used as input.
- The target Radius environment is a local development environment (e.g., using `rad init` with a local Kubernetes cluster) with default Recipes configured for Portable Resources (e.g., a default Redis Recipe that provisions a Redis container).
- Service container images are assumed to be pre-built and available in a local or accessible registry. The conversion does not build images. Image references are emitted as Bicep parameters defaulting to `{project-name}:latest`, overridable at deploy time.
- The PoC targets the reference application shape: two services (webfrontend + apiservice) and two dependencies (Redis + SQL Server). Both dependencies have Radius Portable Resource equivalents (`Applications.Datastores/redisCaches` for Redis, `Applications.Datastores/sqlDatabases` for SQL Server) and are Recipe-backed. More complex topologies are explicitly out of scope.
- Standard service ports (e.g., 8080 for HTTP services, 6379 for Redis, 1433 for SQL Server) are extracted from the `.tmpl.yaml` templates. When templates use `{{ targetPortOrDefault N }}`, the default value `N` is used.
- The PoC is implemented as a subcommand within the existing Radius CLI (`rad`) codebase. This requires access to the Radius CLI source and the ability to build and test the CLI locally.

## Non-Goals

- Full-fidelity conversion of complex Aspire solutions (many services, advanced Azure integrations, custom provisioning).
- Automated round-tripping from Radius back to Aspire.
- Production hardening including security posture, compliance policies, or multi-environment promotion.
- Building container images as part of the conversion process.
- Supporting Aspire solutions that use custom resource types or provisioning hooks beyond the standard template.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer can go from a freshly generated Aspire solution to a deployed Radius application in under 15 minutes (excluding environment setup time), following the documented steps.
- **SC-002**: The deployed Radius application exposes a reachable frontend endpoint that returns a successful response (HTTP 200) when accessed.
- **SC-003**: The frontend service can successfully communicate with the api service through the Radius-defined connection, verified by an end-to-end request that traverses frontend→api.
- **SC-004**: The api service can reach the SQL Server dependency and the Redis dependency (or, if not possible, the limitation is documented, the gap report identifies the specific missing information, and this is validated as a known limitation).
- **SC-005**: The mapping report covers 100% of populated fields in the output app.bicep, with each field traced to a specific Aspire artifact source or marked as an assumption/default.
- **SC-006**: Re-running the conversion on identically regenerated Aspire artifacts produces an app.bicep with byte-for-byte identical content.
- **SC-007**: A developer unfamiliar with the project can read the mapping report and correctly explain, for any given resource in app.bicep, where the data came from — validated by a peer review within 30 minutes of reading the report.
