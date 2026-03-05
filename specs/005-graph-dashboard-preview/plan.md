# Implementation Plan: View Static Bicep Application Graphs in the Radius Dashboard

**Branch**: `005-graph-dashboard-preview` | **Date**: 2026-03-05 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/005-graph-dashboard-preview/spec.md`

## Summary

Add a dedicated `/preview` page to the Radius Dashboard where users can import `ApplicationGraphResponse` JSON (produced by `rad app graph --file --output json`) and visualize it using the existing `AppGraph` React Flow component. Preview graphs are styled distinctly (dashed borders, muted colors, "Preview" banner) to differentiate them from live deployed graphs. The page also supports shareable URLs (compressed data in URL hash fragment) and operates fully offline without any Radius API connection.

## Technical Context

**Language/Version**: TypeScript ~5.2, React 17, Node.js 18/20/21
**Primary Dependencies**: Backstage (Spotify), React Flow v11, Dagre, Material UI v4, pako (compression)
**Storage**: N/A (client-side only, no persistence)
**Testing**: Jest (via backstage-cli), React Testing Library, Playwright (E2E), Storybook v7.6
**Target Platform**: Web browser (Backstage dashboard)
**Project Type**: Web application (Backstage plugin architecture, monorepo with Yarn 4 workspaces)
**Performance Goals**: Graph renders within 5 seconds; interactive at 50 resources / 100 connections
**Constraints**: Fully offline (no Radius API); client-side only; shareable URLs ≤64KB via hash fragment
**Scale/Scope**: Single new page + ~8 new files + ~4 modified files in the dashboard repo

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. API-First Design | PASS | Client-side TypeScript interfaces defined in contracts/. No new server APIs. |
| II. Idiomatic Code Standards | PASS | TypeScript strict mode, ESLint/Prettier, Backstage patterns. |
| III. Multi-Cloud Neutrality | N/A | Dashboard feature, no cloud-specific code. |
| IV. Testing Pyramid Discipline | PASS | Unit tests (Jest/RTL) for all new modules, Storybook stories, Playwright E2E planned. |
| V. Collaboration-Centric | PASS | Enables developers to share architecture previews for PR review. |
| VI. Open Source | PASS | Design spec in design-notes repo; public discussion before implementation. |
| VII. Simplicity Over Cleverness | PASS | Reuses existing AppGraph/ResourceNode components; manual validation over schema library. |
| VIII. Separation of Concerns | PASS | Validation, transformation, URL encoding in separate modules. |
| IX. Incremental Adoption | PASS | New page; no changes to existing live graph workflow. |
| X. TypeScript & React Standards | PASS | Functional components, hooks, Backstage plugin architecture, Storybook stories. |
| XI. Frontend Testing Discipline | PASS | Jest unit tests, Storybook stories for all states, Playwright E2E. |
| XVI. Repository-Specific Standards | PASS | Follows dashboard repo conventions (Yarn 4, backstage-cli, packages/ structure). |
| XVII. Polyglot Coherence | PASS | Uses same `ApplicationGraphResponse` type from Radius API; consistent terminology. |

**Post-design re-check**: All principles still pass. No violations requiring justification.

## Project Structure

### Documentation (this feature)

```text
specs/005-graph-dashboard-preview/
├── plan.md              # This file
├── research.md          # Phase 0: technology research
├── data-model.md        # Phase 1: entity definitions and transformations
├── quickstart.md        # Phase 1: implementation reference
├── contracts/           # Phase 1: TypeScript interface contracts
│   ├── graph-import.md  # Validation & transformation contracts
│   └── shareable-url.md # URL encoding/decoding contracts
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (created by /speckit.tasks)
```

### Source Code (dashboard repository)

```text
dashboard/
├── packages/
│   └── rad-components/
│       └── src/
│           ├── lib/
│           │   ├── graphImport.ts          # NEW: validate, transform, parse
│           │   ├── graphImport.test.ts      # NEW: unit tests
│           │   ├── shareableUrl.ts          # NEW: URL encode/decode
│           │   └── shareableUrl.test.ts      # NEW: unit tests
│           └── components/
│               ├── appgraph/
│               │   └── AppGraph.tsx         # MODIFY: add isPreview prop
│               ├── resourcenode/
│               │   └── ResourceNode.tsx     # MODIFY: preview styling
│               ├── graphimport/
│               │   ├── GraphImportPanel.tsx       # NEW: import UI
│               │   ├── GraphImportPanel.test.tsx   # NEW: unit tests
│               │   └── __docs__/
│               │       └── GraphImportPanel.stories.tsx  # NEW: Storybook
│               └── previewbanner/
│                   ├── PreviewBanner.tsx    # NEW: "Preview" indicator
│                   └── PreviewBanner.test.tsx # NEW: unit tests
├── plugins/
│   └── plugin-radius/
│       └── src/
│           ├── routes.ts                   # MODIFY: add previewRouteRef
│           ├── plugin.ts                   # MODIFY: add PreviewPage extension
│           └── components/
│               └── preview/
│                   ├── PreviewPage.tsx      # NEW: main preview page
│                   └── PreviewPage.test.tsx  # NEW: unit tests
└── packages/
    └── app/
        └── src/
            ├── App.tsx                     # MODIFY: add /preview route
            └── components/
                └── Root/
                    └── Root.tsx            # MODIFY: add sidebar item
```

**Structure Decision**: Frontend-only changes within the existing Backstage monorepo. New code splits across `rad-components` (reusable utilities and components) and `plugin-radius` (page-level integration and routing), following the dashboard's established separation pattern.

## Complexity Tracking

> No constitution violations. No complexity justification needed.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
