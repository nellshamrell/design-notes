# Feature Specification: View Static Bicep Application Graphs in the Radius Dashboard

**Feature Branch**: `005-graph-dashboard-preview`
**Created**: 2026-03-04
**Status**: Draft
**Input**: User description: "View Static Bicep Application Graphs in the Radius Dashboard"

## Clarifications

### Session 2026-03-04

- Q: Which term should be the canonical UI label — "Preview", "Static", or "File"? → A: "Preview" — used consistently everywhere (banner, badges, labels, docs).
- Q: When the user imports new data while a graph is already displayed, should the dashboard silently replace, prompt for confirmation, or show side-by-side? → A: Silent replace — new import immediately replaces the current graph with no confirmation prompt.
- Q: What should happen when a graph exceeds the performance threshold (50+ resources)? → A: Warn then render — display a warning (e.g., "Large graph — performance may be affected") but proceed with rendering.
- Q: What should happen when graph data is too large to encode in a shareable URL? → A: Notify with fallback — show a message that the graph is too large to share via URL and suggest exporting the JSON file instead.
- Q: How should the preview graph import feature integrate with the dashboard navigation? → A: Dedicated page — a separate page/route accessible from the main navigation, with its own import controls.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Import JSON Graph Data (Priority: P1)

A developer has already run the Radius CLI command to produce an application graph in JSON format from a Bicep file. They open the Radius Dashboard and import this JSON data — either by pasting it into a text area or uploading a `.json` file. The dashboard parses the data, validates it against the expected application graph schema, and renders the graph visually. The graph clearly indicates that it represents a preview (not a deployed application). This allows the developer to visualize their planned application topology without needing a running Radius control plane.

**Why this priority**: This is the foundational capability. It requires no compilation tooling, works fully offline, and delivers immediate value by letting users visualize graph data they already have. Every other story builds on this graph rendering capability.

**Independent Test**: Can be fully tested by pasting valid JSON graph output into the dashboard and verifying the graph renders correctly. Delivers immediate visualization value with zero infrastructure.

**Acceptance Scenarios**:

1. **Given** the dashboard is open with no Radius API connection, **When** the user pastes valid application graph JSON into the import area, **Then** the dashboard renders a visual graph showing all resources and connections.
2. **Given** the dashboard is open, **When** the user uploads a `.json` file containing valid application graph data, **Then** the graph renders identically to pasted JSON input.
3. **Given** the dashboard is open, **When** the user pastes malformed or invalid JSON, **Then** the dashboard displays a clear, human-readable error message indicating what is wrong.
4. **Given** the dashboard is open, **When** the user pastes JSON that does not conform to the application graph schema (e.g., missing required fields), **Then** the dashboard displays a validation error specifying which fields are missing or invalid.
5. **Given** the user has imported valid graph data, **When** the graph renders, **Then** all resources show a "Not Deployed" status indicator and the graph is labeled as a "Preview."

---

### User Story 2 - Visual Distinction Between Preview and Live Graphs (Priority: P2)

A developer is using the Radius Dashboard and may be viewing both deployed application graphs (from the Radius API) and preview graphs (from imported data). The dashboard must make it visually obvious which type of graph is being viewed so the developer never confuses a planned topology with the actual deployed state. Preview graphs use distinct visual styling — such as dashed borders on resource nodes, dashed connection lines, and a banner or badge indicating "Preview" status.

**Why this priority**: Without clear visual distinction, users could mistake a preview graph for a live deployment, leading to confusion and potentially incorrect decisions. This is critical for trust and usability once the import capability (P1) exists.

**Independent Test**: Can be tested by comparing the rendering of a preview graph against a live graph side by side, verifying all visual distinction cues are present and unambiguous.

**Acceptance Scenarios**:

1. **Given** a preview graph is rendered from imported data, **When** the user views it, **Then** a prominent banner or badge displays "Preview" in the graph header area.
2. **Given** a preview graph is rendered, **When** the user inspects resource nodes, **Then** each node displays "Not Deployed" status with visually distinct styling (e.g., dashed borders, muted colors) compared to live resource nodes.
3. **Given** a preview graph is rendered, **When** the user views connections between resources, **Then** connections are styled distinctly (e.g., dashed lines) to indicate they are not yet active.
4. **Given** a user navigates between a live graph and a preview graph, **When** they view each, **Then** the visual differences are immediately noticeable without needing to read labels.

---

### User Story 3 - Shareable Graph URL (Priority: P3)

A team lead wants to share a preview graph with teammates for discussion during a PR review or architecture planning session. After importing or uploading a graph, the dashboard generates a shareable URL that encodes the graph data. Opening this URL in a browser renders the graph without requiring the recipient to re-upload or re-import data. All data remains client-side — no server-side storage is required.

**Why this priority**: Sharing is a collaboration enhancement that builds on all prior capabilities. It is valuable but not essential for the core visualization workflow.

**Independent Test**: Can be tested by importing a graph, generating the shareable URL, opening it in a new browser window, and verifying the graph renders identically.

**Acceptance Scenarios**:

1. **Given** a preview graph is rendered in the dashboard, **When** the user clicks a "Share" or "Copy Link" action, **Then** a URL is generated and copied to the clipboard.
2. **Given** a shareable URL has been generated, **When** another user opens it in their browser, **Then** the graph renders identically without any file upload or import required.
3. **Given** a shareable URL is opened, **When** the graph data encoded in the URL is corrupted or truncated, **Then** the dashboard displays a clear error message indicating the shared link is invalid.
4. **Given** a graph has been imported, **When** the shareable URL is generated, **Then** no data is sent to or stored on any server — all graph data is encoded in the URL itself.

---

### Edge Cases

- What happens when the user uploads an empty file (0 bytes)?
- What happens when the imported JSON describes a graph with zero resources?
- What happens when the graph contains circular connections (resource A connects to B, B connects to A)?
- How does the dashboard handle very large graphs (e.g., 100+ resources with many connections)? A warning is displayed but the graph is still rendered.
- What happens when the user tries to import while a previous graph is already displayed? The new graph silently replaces the old one with no confirmation.
- What happens when the shareable URL exceeds browser URL length limits due to large graph data? The dashboard displays a message that the graph is too large to share via URL and suggests exporting the JSON file instead.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The dashboard MUST provide a dedicated preview page, accessible from the main navigation, for importing and viewing preview graphs.
- **FR-002**: The dashboard MUST accept application graph data via JSON paste into a text input area on the preview page.
- **FR-003**: The dashboard MUST accept application graph data via `.json` file upload on the preview page.
- **FR-004**: The dashboard MUST validate imported JSON against the application graph schema and display specific, human-readable error messages for invalid input.
- **FR-005**: The dashboard MUST render imported graph data as a visual graph showing resources as nodes and connections as edges.
- **FR-006**: The dashboard MUST reuse the same graph visualization component for both preview graphs and live deployed graphs.
- **FR-007**: The dashboard MUST display a "Preview" indicator (banner, badge, or label) on graphs generated from imported data.
- **FR-008**: The dashboard MUST style preview graph resource nodes with a "Not Deployed" status and visually distinct appearance (e.g., dashed borders, muted colors).
- **FR-009**: The dashboard MUST style preview graph connections distinctly (e.g., dashed lines) to differentiate them from live connections.
- **FR-010**: The dashboard MUST generate a shareable URL that encodes the current preview graph data entirely in the URL (no server-side storage).
- **FR-011**: The dashboard MUST render a graph correctly when opened via a shareable URL without requiring any additional user action.
- **FR-012**: When the graph data exceeds the maximum encodable URL size, the dashboard MUST display a clear message informing the user that the graph is too large to share via URL and MUST suggest exporting the JSON file as a fallback.
- **FR-013**: The preview graph viewing workflow MUST function without any Radius API or control plane connection.
- **FR-014**: The dashboard MUST handle empty graphs (zero resources) gracefully, displaying an appropriate message rather than a blank view.
- **FR-015**: The dashboard MUST silently replace the currently displayed graph when new data is imported, with no confirmation prompt and no stale state retained.
- **FR-016**: The dashboard MUST display a warning message (e.g., "Large graph — performance may be affected") when the imported graph exceeds 50 resources or 100 connections, but MUST still render the graph.

### Key Entities

- **Application Graph**: The top-level data structure representing an application's topology. Contains a collection of resources and their connections. Corresponds to the output of the Radius CLI's graph command.
- **Resource**: A node in the application graph representing a Radius resource (e.g., container, database, gateway). Has a name, type, provisioning state, output resources, and connections. For preview graphs, provisioning state is always "Not Deployed" and output resources are empty.
- **Connection**: A relationship between two resources, representing a dependency or communication path. Has a name, target resource identifier, and direction (inbound or outbound).
- **Graph Source**: Metadata indicating how the graph was created — either "live" (from the Radius API for a deployed application) or "preview" (from imported file/JSON data). Determines visual styling.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can import JSON graph data and see the rendered graph within 5 seconds of submission.
- **SC-002**: 95% of users can successfully import and visualize a graph on their first attempt without consulting documentation.
- **SC-003**: Users can visually distinguish between a preview graph and a live graph within 2 seconds of viewing, without reading text labels.
- **SC-004**: Graphs with up to 50 resources and 100 connections render and remain interactive (pan, zoom) without noticeable lag.
- **SC-005**: Shareable URLs successfully render the encoded graph for recipients 100% of the time when the URL is not corrupted or truncated.
- **SC-006**: The preview graph workflow functions fully without any network connection to a Radius control plane.

## Assumptions

- The Radius CLI's `rad app graph --file --output json` command is stable and its output format (`ApplicationGraphResponse`) is versioned or backward-compatible.
- The dashboard's existing graph visualization component can be extended with additional styling options (dashed borders, dashed lines, color variations) without a full rewrite.
- Graph data encoded in shareable URLs will be compressed to stay within typical browser URL length limits (~2,000 characters) for moderately sized graphs. When graphs exceed this limit, the share action will notify the user and suggest JSON export as a fallback.
- The application graph data model is sufficient to represent all resource types and connections that Radius supports in Bicep files.

## Out of Scope

- Exporting graphs in Graphviz DOT format (may be added in a future iteration)
- Deploying applications from the dashboard
- Uploading and compiling Bicep files directly in the dashboard (users should use the CLI to generate JSON graph data)
- Editing Bicep files within the dashboard
- Real-time updates or file-watch mode for Bicep files
- Multi-file Bicep projects beyond what the CLI handles as a single compilation unit
- Authentication or authorization for shared graph URLs
- Server-side storage of uploaded or shared graph data
- Comparing two graphs side by side (preview vs. live, or two versions of a preview graph)
