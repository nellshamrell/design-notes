# Research: View Static Bicep Application Graphs in the Radius Dashboard

**Date**: 2026-03-05
**Feature**: [spec.md](spec.md) | [plan.md](plan.md)

## Research Questions & Findings

### RQ1: Dashboard Technology Stack

**Decision**: The dashboard is a Backstage-based application using React 17, TypeScript ~5.2, Material UI v4, and Yarn 4 workspaces.

**Rationale**: This is not a choice — it's the existing stack. All new code must integrate with these technologies and follow Backstage plugin architecture patterns.

**Findings**:

| Layer | Technology |
|-------|-----------|
| Framework | Backstage (Spotify developer portal) |
| Frontend | React 17, TypeScript ~5.2, Material UI v4 |
| Build | `backstage-cli`, TypeScript compiler |
| Package manager | Yarn 4.0.2 (Berry) with workspaces |
| Node.js | 18, 20, or 21 |
| Testing | Jest (via backstage-cli), Playwright (E2E), React Testing Library |
| Storybook | v7.6 (in `rad-components`) |
| Linting | ESLint + Prettier (Spotify config) |

**Monorepo structure**:
- `packages/app/` — main Backstage app shell, routing, sidebar
- `packages/backend/` — Backstage backend
- `packages/rad-components/` — shared Radius UI components (graph rendering lives here)
- `plugins/plugin-radius/` — Radius Backstage plugin (pages, API clients, routing)

### RQ2: Graph Visualization Library

**Decision**: Use React Flow (v11) + Dagre for graph rendering — these are already in use.

**Rationale**: The existing `AppGraph` component in `rad-components` already uses React Flow for interactive rendering and Dagre for automatic layout. Reusing these avoids adding new dependencies and ensures visual consistency.

**Findings**:
- `reactflow` v11.10.1 — interactive graph nodes, edges, pan/zoom, controls
- `@dagrejs/dagre` v1.0.4 + `dagre` v0.8.5 — automatic hierarchical layout (top-to-bottom)
- Both already in `packages/rad-components/package.json`
- The `AppGraph.tsx` component converts data into React Flow `Node[]` and `Edge[]`, applies Dagre layout, and renders with `<ReactFlow>`
- `ResourceNode.tsx` is a custom node type showing resource name and type
- React Flow supports custom node styles (borders, colors, opacity), edge styles (dashed, animated, colors), and overlay elements (banners) — all needed for preview distinction

### RQ3: Live Graph Data Flow

**Decision**: The preview workflow will bypass the live graph data flow entirely. No Kubernetes API or Radius API needed.

**Rationale**: The live graph flow depends on `kubernetesApi.proxy()` to reach a running Radius control plane. The preview feature operates on local JSON data, so it bypasses this entirely.

**Live data flow** (for reference):
1. `ApplicationTab` component calls `kubernetesApi.proxy()` directly (bypasses `RadiusApi` abstraction)
2. POST to `/apis/api.ucp.dev/v1alpha3/{applicationId}/getGraph?api-version=2023-10-01-preview`
3. Proxied through Backstage backend → Kubernetes cluster → Radius control plane
4. Response is `ApplicationGraphResponse` JSON
5. Passed to `<AppGraph graph={data} />` for rendering

**Important**: The `ApplicationTab` bypasses the `RadiusApi` class and calls the Kubernetes proxy directly. The `RadiusApi` does not have a `getGraph` method.

### RQ4: Data Model Mapping

**Decision**: The preview feature must accept the Go API's `ApplicationGraphResponse` JSON format and transform it to the dashboard's `AppGraph` TypeScript type for rendering.

**Rationale**: Users produce JSON via the CLI (`rad app graph --file --output json`), which outputs `ApplicationGraphResponse`. The `AppGraph` React component expects the dashboard's `AppGraph` type. A transformation layer is needed.

**Key differences between Go API type and Dashboard type**:

| Field | Go `ApplicationGraphResponse` | Dashboard `AppGraph` |
|-------|-------------------------------|---------------------|
| Top-level `name` | Not present | `name: string` |
| Resource.`provider` | Not present | `provider: string` |
| Resource.`outputResources` | `OutputResources[]` (separate type) | `resources?: Resource[]` (recursive, renamed) |
| Resource.`connections` | Required (never null) | `connections?: Connection[]` (optional) |
| Connection.`name` | Not present | `name: string` |
| Connection.`type` | Not present | `type: string` |
| Connection.`provider` | Not present | `provider: string` |

**Transformation needed**:
- Add top-level `name` (derive from first application resource or use "Preview")
- Map `outputResources` → `resources` (nested)
- Derive `provider` from resource `type` (extract namespace before `/`)
- Derive connection `name`, `type`, `provider` by looking up the target resource by `id`
- Default optional fields to `[]` when missing

### RQ5: Offline/Mock Data Support

**Decision**: No existing offline mode exists. The preview page will be the first purely client-side feature in the dashboard.

**Rationale**: All current dashboard features require a live Kubernetes cluster with Radius installed. The preview page introduces a new pattern: client-side-only rendering with no API dependencies.

**Existing patterns**:
- `sampledata.ts` provides a `DemoApplication` fixture for tests and Storybook stories only
- No runtime mock mode or feature flag for offline operation
- The preview page can follow the Storybook pattern of passing data directly to `<AppGraph>`, but at the route level

### RQ6: Routing and Navigation Integration

**Decision**: Add a new route `/preview` to the Backstage app with a sidebar entry, following existing patterns.

**Rationale**: The spec requires a dedicated page. Backstage uses `FlatRoutes` in `App.tsx` and `SidebarItem` in `Root.tsx`. A new route ref + routable extension + sidebar entry follows established dashboard conventions.

**Existing patterns**:
- Route refs defined in `plugins/plugin-radius/src/routes.ts`
- Routable extensions created in `plugins/plugin-radius/src/plugin.ts` via `createRoutableExtension`
- URL paths mapped in `packages/app/src/App.tsx` via `<Route path="..." element={...} />`
- Sidebar entries in `packages/app/src/components/Root/Root.tsx`
- The new page needs: route ref, routable extension, route mapping, sidebar item

### RQ7: DOT Export Format

**Decision**: Replicate the CLI's DOT format in TypeScript for client-side export.

**Rationale**: The spec requires DOT export consistent with `rad app graph --output dot`. The format is straightforward text generation that can be reimplemented in TypeScript without porting Go code.

**DOT format details** (from `display_dot.go`):
- `digraph "name" { ... }` wrapper
- `rankdir=LR` (left-to-right layout)
- Node font: Helvetica, filled style
- Radius resources: `shape=box, fillcolor=lightblue`
- Non-Radius resources: `shape=ellipse, fillcolor=lightyellow`
- Node labels: `"name\n(type)"`
- Edges: outbound connections only, deduplicated, sorted
- Deterministic output: resources sorted by type then name

### RQ8: Shareable URL Encoding

**Decision**: Use URL hash fragment with Base64-encoded, compressed JSON.

**Rationale**: Hash fragments are not subject to the ~2,000 character URL query string limit and are not sent to the server. Using `pako` (deflate compression) + Base64 encoding can compress typical graph JSON significantly. For very large graphs that still exceed practical limits (~64KB for most browsers), show the fallback message per spec.

**Alternatives considered**:
- Query parameters: Limited to ~2,000 chars, sent to server — rejected
- `encodeURIComponent` only: No compression, URL too large for moderate graphs — rejected
- IndexedDB + short ID: Requires same browser, no cross-device sharing — rejected

### RQ9: JSON Validation Approach

**Decision**: Validate imported JSON using a TypeScript validation function that checks required fields and types against the `ApplicationGraphResponse` schema.

**Rationale**: A lightweight validation function is simpler than adding a JSON Schema validation library. The schema is small (3 types, ~10 fields total) and unlikely to change frequently.

**Alternatives considered**:
- JSON Schema with `ajv`: Adds dependency, overkill for this small schema — rejected
- `zod` schema: Good TypeScript integration but adds dependency — acceptable alternative
- Manual validation: Simple, no dependencies, sufficient for 3 types — chosen
