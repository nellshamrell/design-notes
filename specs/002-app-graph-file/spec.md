# Feature Specification: `rad app graph --file` Static Graph from Bicep

**Feature Branch**: `002-app-graph-file`
**Created**: 2026-03-03
**Status**: Draft
**Input**: User description: "Add a --file flag to the rad app graph CLI command that generates an application graph directly from a Bicep file, without requiring a deployed Radius environment"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Preview Application Graph from Bicep File (Priority: P1)

As a developer authoring a Radius application in Bicep, I want to run `rad app graph --file app.bicep` and see the resource topology and connections for my application before deploying, so I can validate my architecture at design time without needing a live Radius environment.

**Why this priority**: This is the core value proposition of the feature. Without it, developers must deploy to a live environment just to see their application graph, which slows down the inner development loop.

**Independent Test**: Can be fully tested by creating a Bicep file with multiple interconnected Radius resources and running `rad app graph --file <path>`. Delivers immediate value by showing the resource graph offline.

**Acceptance Scenarios**:

1. **Given** a valid Bicep file containing an application with containers and connections using literal URL sources (e.g., `http://backend:3000`), **When** the user runs `rad app graph --file app.bicep`, **Then** the command displays a text tree showing all resources, their types, and connection relationships (inbound/outbound) in the same format as the existing live `rad app graph` output.
2. **Given** a valid Bicep file containing resources with connections that reference other resources by ID (e.g., `redis.id` in Bicep), **When** the user runs `rad app graph --file app.bicep`, **Then** the command resolves the compiled ARM expression `[reference('redis').id]` to the correct resource and displays the connection.
3. **Given** a valid Bicep file, **When** the user runs `rad app graph --file app.bicep`, **Then** all resources show a provisioning state of `NotDeployed` and an empty list of output resources.
4. **Given** a pre-compiled ARM JSON template file, **When** the user runs `rad app graph --file template.json`, **Then** the command produces the same graph output as it would from the equivalent Bicep source.

---

### User Story 2 - Mutual Exclusivity of `--file` and App Name (Priority: P1)

As a CLI user, I want clear feedback when I mistakenly provide both `--file` and an application name argument, so I understand these are separate modes of operation.

**Why this priority**: Incorrect usage must produce a clear error to prevent confusion. This is essential for a usable CLI experience and is tightly coupled to the core feature.

**Independent Test**: Can be tested by invoking the command with conflicting arguments and verifying the error message.

**Acceptance Scenarios**:

1. **Given** the user runs `rad app graph myapp --file app.bicep`, **When** the command processes arguments, **Then** it exits with a clear error message stating that `--file` and the positional application name argument are mutually exclusive.
2. **Given** the user runs `rad app graph --file app.bicep` without a positional app name, **When** the command runs, **Then** it proceeds to generate the static graph without error.
3. **Given** the user runs `rad app graph myapp` without `--file`, **When** the command runs, **Then** it proceeds with the existing live graph behavior unchanged.

---

### User Story 3 - Graceful Handling of Unresolvable Connections (Priority: P2)

As a developer whose Bicep file contains connections with parameterized or expression-based sources that cannot be statically resolved, I want the tool to show a partial graph with warnings rather than failing entirely, so I still get value from the resources and connections it can resolve.

**Why this priority**: Real-world Bicep files often use parameters and expressions. Graceful degradation ensures the feature is useful even when not all connections can be resolved.

**Independent Test**: Can be tested by creating a Bicep file with parameterized connection sources and verifying partial output plus warning messages.

**Acceptance Scenarios**:

1. **Given** a Bicep file where a connection source uses a parameter value (e.g., `source: someParam`), **When** the user runs `rad app graph --file app.bicep`, **Then** the graph is displayed with all resolvable resources and connections, and a warning is emitted to stderr indicating which connection could not be resolved and why.
2. **Given** a Bicep file with conditional resources (using `if` in Bicep), **When** the user runs `rad app graph --file app.bicep`, **Then** the conditional resources are included in the graph with a warning that they may not be deployed depending on conditions.

---

### User Story 4 - Offline Operation Without Radius Environment (Priority: P2)

As a developer working in an environment without access to a Radius control plane (e.g., on an airplane, in a CI pipeline for validation), I want `rad app graph --file` to work completely offline with no API calls to any Radius environment.

**Why this priority**: Offline support is a key differentiator of this feature over the existing live command and enables new workflows like CI-based architecture validation.

**Independent Test**: Can be tested by disconnecting from the network (or ensuring no Radius environment is configured) and running the command with a valid Bicep file, verifying it produces output without errors.

**Acceptance Scenarios**:

1. **Given** no Radius environment is configured or reachable, **When** the user runs `rad app graph --file app.bicep`, **Then** the command succeeds and produces the application graph without attempting any network calls to a Radius control plane.
2. **Given** the Bicep compiler (`rad-bicep`) is not installed locally, **When** the user runs `rad app graph --file app.bicep`, **Then** the command auto-downloads the compiler (matching existing CLI behavior) before proceeding.
3. **Given** a valid Bicep file, **When** the user runs `rad app graph --file app.bicep --output json`, **Then** the command produces machine-readable JSON output of the application graph, suitable for consumption by CI pipelines or other tooling.

---

### User Story 5 - Handling Files with Multiple or No Application Resources (Priority: P3)

As a developer whose Bicep file may not declare an explicit application resource, or declares multiple applications, I want the graph command to handle these cases sensibly.

**Why this priority**: While most Bicep files will have a single application, edge cases around zero or multiple applications need defined behavior to avoid confusing results.

**Independent Test**: Can be tested by creating Bicep files with zero or multiple application resources and verifying the output behavior.

**Acceptance Scenarios**:

1. **Given** a Bicep file with no explicit `Applications.Core/applications` resource, **When** the user runs `rad app graph --file app.bicep`, **Then** all Radius resources in the file are treated as part of a single implicit application and the graph is displayed.
2. **Given** a Bicep file with exactly one `Applications.Core/applications` resource, **When** the user runs `rad app graph --file app.bicep`, **Then** only resources whose `application` property references that application are included in the graph.
3. **Given** a Bicep file with two or more `Applications.Core/applications` resources, **When** the user runs `rad app graph --file app.bicep`, **Then** the command exits with an error message indicating multiple applications were found and asking the user to specify which one.

---

### User Story 6 - Visual Graph Output via Graphviz DOT Format (Priority: P2)

As a developer reviewing my application architecture, I want to generate a visual diagram of the application graph from my Bicep file, so I can share it in documentation, design reviews, or presentations without manually drawing the topology.

**Why this priority**: Text output is useful for quick terminal checks, but visual diagrams are essential for documentation and team communication. DOT format is widely supported by Graphviz, VS Code extensions, and online renderers.

**Independent Test**: Run `rad app graph --file app.bicep --output dot` and pipe the output to `dot -Tpng -o graph.png` to produce a PNG image, or paste into an online Graphviz renderer.

**Acceptance Scenarios**:

1. **Given** a valid Bicep file containing an application with containers and connections, **When** the user runs `rad app graph --file app.bicep --output dot`, **Then** the command outputs a valid Graphviz DOT language string to stdout representing the application graph, with nodes for each resource (labeled with name and type) and directed edges for each connection.
2. **Given** a valid Bicep file, **When** the user runs `rad app graph --file app.bicep --output dot | dot -Tpng -o graph.png`, **Then** Graphviz successfully renders the DOT output to a PNG image without errors.
3. **Given** a Bicep file with non-Radius resources, **When** the user runs `rad app graph --file app.bicep --output dot`, **Then** non-Radius resources appear as distinctly styled nodes (different shape or color) in the DOT output.
4. **Given** the existing live `rad app graph` command, **When** the user runs `rad app graph myapp --output dot`, **Then** the DOT output format is also available in live mode (not limited to `--file` mode).

---

### Edge Cases

- What happens when the `--file` path does not exist or is not readable? The command exits with an error message indicating the file was not found or could not be read.
- What happens when the Bicep file has syntax errors? The Bicep compilation step fails and the command displays the compiler's error output to the user.
- What happens when the file contains no Radius resources? The command produces an empty graph with a message indicating no Radius resources were found.
- What happens when the file contains Bicep module references? Resources inside modules compiled as nested deployments are not traversed; a warning is emitted noting that module resources are not included in the static graph.
- What happens when a connection source uses `format()` or other non-reference ARM expressions? The connection is skipped with a warning to stderr.
- What happens when resources have duplicate names? Each resource is uniquely identified by its synthesized resource ID (based on type and name), so duplicates of the same type and name would collide. The command should warn about duplicate resources.
- What happens when the file contains non-Radius resources (e.g., Azure or Kubernetes resources)? They are included in the graph as nodes showing their type and name, but without Radius-specific properties like connections or routes. This gives developers visibility into the full template topology.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The CLI MUST accept a `--file` flag on the `rad app graph` command that takes a file path to a `.bicep` or `.json` (ARM template) file.
- **FR-002**: The `--file` flag and the positional `<app-name>` argument MUST be mutually exclusive. Providing both MUST result in a clear error message.
- **FR-003**: When `--file` is provided with a `.bicep` file, the system MUST compile it to ARM JSON using the existing Bicep compilation pipeline (e.g., `PrepareTemplate()`).
- **FR-004**: When `--file` is provided with a `.json` file, the system MUST parse it directly as an ARM JSON template without compilation.
- **FR-005**: The system MUST extract all Radius resources from the ARM JSON template's `resources` map, reading each resource's type (with API version stripped), name (from `properties.name`), and Radius properties (from `properties.properties`).
- **FR-006**: The system MUST generate a synthetic Radius resource ID for each extracted resource using the pattern `/planes/radius/local/resourceGroups/default/providers/{Type}/{Name}`.
- **FR-007**: The system MUST extract `connections` and `routes` from each resource's Radius properties and resolve them to build the application graph.
- **FR-008**: The system MUST resolve ARM expressions of the form `[reference('symbolicName').id]` by looking up the symbolic name in the template's resources and using the target resource's synthesized ID.
- **FR-009**: The system MUST pass literal URL connection sources (e.g., `http://backend:3000`) through to the graph builder, allowing hostname-to-resource matching.
- **FR-010**: The system MUST handle unresolvable ARM expressions (parameters, `format()`, complex expressions) gracefully by skipping the connection and emitting a warning to stderr.
- **FR-011**: The system MUST set `provisioningState` to `NotDeployed` for all resources in the static graph.
- **FR-012**: The system MUST set `outputResources` to an empty list for all resources in the static graph.
- **FR-013**: The system MUST produce output in the same text format as the existing `rad app graph` command.
- **FR-014**: The system MUST sort all resources and connections deterministically for reproducible output.
- **FR-015**: When a Bicep file contains exactly one application resource, the system MUST scope the graph to resources referencing that application. When no application resource exists, the system MUST treat all resources as part of a single implicit application. When the file contains multiple application resources, the system MUST exit with an error message indicating that multiple applications were found and asking the user to specify which one (a future `--application` flag may address this).
- **FR-016**: The system MUST emit warnings to stderr for conditional resources, module references, and any other constructs that may affect graph completeness.
- **FR-017**: The system MUST operate entirely offline when `--file` is used, making no API calls to a Radius control plane.
- **FR-018**: The system MUST auto-download the Bicep compiler if not already present, matching existing CLI auto-download behavior.
- **FR-019**: The `--file` mode MUST support `--output json` to produce machine-readable JSON output of the application graph, using the same JSON structure as the existing live `rad app graph --output json` command. The default output format remains the text tree.
- **FR-020**: The system MUST include non-Radius resources (e.g., `Microsoft.Storage/storageAccounts`, Kubernetes resources) found in the Bicep file as nodes in the graph. These nodes MUST display the resource type and name but are not expected to have Radius-specific properties (connections, routes, application reference). They provide visibility into the full resource topology declared in the file.
- **FR-021**: The system MUST support `--output dot` to produce a Graphviz DOT language representation of the application graph. The DOT output MUST include a node for each resource (labeled with resource name and type), a directed edge for each outbound connection (from source to target), and visually distinguish non-Radius resources from Radius resources using different node shapes. The DOT output MUST be valid input for the `dot` command-line tool. This output format MUST be available in both `--file` mode and live mode.

### Key Entities

- **Bicep/ARM Resource**: A declared resource in the Bicep or ARM JSON template. Key attributes: symbolic name, resource type, resource name, Radius properties (connections, routes, application reference).
- **Synthesized Resource ID**: A generated Radius resource ID constructed from resource type and name, using a default resource group placeholder. Used as the unique key for graph nodes.
- **Application Graph**: The output structure containing resource nodes and their connection edges (inbound, outbound). Produced by the graph computation logic.
- **Connection**: A named relationship from one resource to another, defined via a source (resource ID or URL). Connections have direction (inbound/outbound).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Developers can visualize the complete resource topology of a Bicep-defined Radius application without deploying, in under 5 seconds for a typical application file (10-20 resources).
- **SC-002**: The static graph output matches the format and structure of the existing live `rad app graph` output, so developers can read it using the same mental model.
- **SC-003**: 100% of literal URL connections and intra-template resource ID references are correctly resolved and displayed in the graph.
- **SC-004**: Unresolvable connections produce clear warning messages that identify the affected resource and connection name, enabling the developer to understand what is missing.
- **SC-005**: The command produces identical output when run multiple times against the same input file (deterministic output).
- **SC-006**: The command succeeds with no network connectivity when the Bicep compiler is already available locally.
- **SC-007**: CI pipelines can consume the graph programmatically via `--output json`, receiving a structured JSON response with the same schema as the existing live graph JSON output.
- **SC-008**: Developers can generate a visual diagram of their application graph by piping `--output dot` to Graphviz (`dot -Tpng`), producing a valid PNG/SVG image suitable for documentation and design reviews.

## Clarifications

### Session 2026-03-03

- Q: What should the command do when a Bicep file contains multiple `Applications.Core/applications` resources? → A: Exit with an error asking the user to specify which application (future `--application` filter).
- Q: Should `--file` mode support `--output json` for machine-readable output (e.g., CI pipelines)? → A: Yes, support `--output json` in `--file` mode from the start.
- Q: How should the command handle non-Radius resources (e.g., Azure, Kubernetes) present in a Bicep file? → A: Include non-Radius resources in the graph as untyped nodes for visibility.
- Q: Should users be able to visualize the graph as a diagram? → A: Yes, support `--output dot` to produce Graphviz DOT format. This is widely supported by tools (Graphviz CLI, VS Code extensions, online renderers) and avoids adding a rendering dependency to the CLI itself.

## Assumptions

- The Bicep compiler (`rad-bicep`) is available for local invocation or can be auto-downloaded using the existing CLI mechanism.
- Most Radius Bicep files declare a single application resource. Multi-application files are uncommon.
- Bicep module support is out of scope; users will be informed via warnings.
- Parameter file support (`--parameters`) is out of scope for this feature; parameterized values will produce warnings rather than errors.
- The existing `computeGraph()` function and output rendering logic can be reused or adapted without significant changes.
- A default placeholder resource group (`default`) is acceptable for synthesized resource IDs since these IDs are only used as internal graph keys and not for actual API calls.

## Scope Boundaries

### In Scope

- Single Bicep files with inline resources
- Pre-compiled ARM JSON template files
- Literal URL connection sources
- ARM `[reference('X').id]` expression resolution within the same template
- All Radius resource types (core and portable)
- Non-Radius resources included as untyped graph nodes for visibility
- Gateway route extraction
- Graphviz DOT output format for visual graph rendering

### Out of Scope (Future Work)

- Recursive processing of Bicep modules / nested ARM templates
- Full ARM template expression evaluation engine
- Resolving recipe-provisioned output resources
- Comparing static graph vs. live deployed graph (diff mode)
- Parameter file support (`--parameters`)
- Support for multi-file Bicep projects beyond single-file compilation
