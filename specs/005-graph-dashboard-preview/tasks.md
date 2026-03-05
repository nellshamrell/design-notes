# Tasks: View Static Bicep Application Graphs in the Radius Dashboard

**Input**: Design documents from `/specs/005-graph-dashboard-preview/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅, quickstart.md ✅

**Tests**: Tests are included — the spec explicitly calls for Jest unit tests, Storybook stories, and Playwright E2E tests. Tests are written alongside or after implementation per Backstage conventions.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- Includes exact file paths relative to the `dashboard/` repository root

## Path Conventions

- **Shared components/libs**: `packages/rad-components/src/`
- **Plugin pages/routing**: `plugins/plugin-radius/src/`
- **App shell/routing**: `packages/app/src/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Install dependencies and create project scaffolding

- [ ] T001 Add `pako` and `@types/pako` dependencies to `packages/rad-components/package.json` and run `yarn install`
- [ ] T002 [P] Create directory structure for new components: `packages/rad-components/src/components/graphimport/`, `packages/rad-components/src/components/previewbanner/`, `packages/rad-components/src/components/graphactions/`, `packages/rad-components/src/components/graphimport/__docs__/`, and `plugins/plugin-radius/src/components/preview/`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core utility modules that ALL user stories depend on — validation, transformation, and TypeScript interfaces

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T003 Define `ApplicationGraphResponse`, `ApplicationGraphResource`, `ApplicationGraphOutputResource`, and `ApplicationGraphConnection` TypeScript interfaces in `packages/rad-components/src/lib/graphImport.ts` per contracts/graph-import.md input schema
- [ ] T004 Implement `validateApplicationGraphResponse(data: unknown): ValidationResult<ApplicationGraphResponse>` in `packages/rad-components/src/lib/graphImport.ts` per validation rules V-001 through V-010 from contracts/graph-import.md — must report ALL errors, not just the first
- [ ] T005 Implement `transformToAppGraph(response: ApplicationGraphResponse, name?: string): AppGraph` in `packages/rad-components/src/lib/graphImport.ts` per transformation rules from contracts/graph-import.md and data-model.md — derive `provider` from `type.split('/')[0]`, enrich connections by looking up target resources, handle missing references gracefully
- [ ] T006 Implement `parseGraphJson(text: string): ValidationResult<ApplicationGraphResponse>` in `packages/rad-components/src/lib/graphImport.ts` — combines `JSON.parse` + `validateApplicationGraphResponse`, returns `"Invalid JSON: {message}"` on parse error
- [ ] T007 Write unit tests for all validation, transformation, and parse functions in `packages/rad-components/src/lib/graphImport.test.ts` — cover: valid input, empty resources, missing fields, malformed JSON, unknown connection targets, output resource mapping, provider derivation

**Checkpoint**: Foundation ready — validation and transformation pipeline is tested and available for all user stories

---

## Phase 3: User Story 1 — Import JSON Graph Data (Priority: P1) 🎯 MVP

**Goal**: Users can paste or upload `ApplicationGraphResponse` JSON and see the graph rendered in the dashboard on a dedicated `/preview` page

**Independent Test**: Paste valid JSON graph output into the dashboard preview page and verify the graph renders correctly with all resources and connections visible

### Implementation for User Story 1

- [ ] T008 [P] [US1] Create `GraphImportPanel` component in `packages/rad-components/src/components/graphimport/GraphImportPanel.tsx` — text area for JSON paste, file upload input (`.json` files), error display for validation failures, loading state; calls `parseGraphJson` on submit and `onImport(response)` callback on success
- [ ] T009 [P] [US1] Write unit tests for `GraphImportPanel` in `packages/rad-components/src/components/graphimport/GraphImportPanel.test.tsx` — test paste valid JSON renders success callback, paste invalid JSON shows error messages, file upload triggers parse, empty input handling
- [ ] T010 [P] [US1] Create Storybook stories for `GraphImportPanel` in `packages/rad-components/src/components/graphimport/__docs__/GraphImportPanel.stories.tsx` — stories: Empty state, With validation error, With loaded data
- [ ] T011 [US1] Add `previewPageRouteRef` to `plugins/plugin-radius/src/routes.ts` — create route ref with id `'radius-preview-page'`
- [ ] T012 [US1] Add `PreviewPage` routable extension to `plugins/plugin-radius/src/plugin.ts` — lazy-load `PreviewPage` component, bind to `previewPageRouteRef`
- [ ] T013 [US1] Create `PreviewPage` component in `plugins/plugin-radius/src/components/preview/PreviewPage.tsx` — renders `GraphImportPanel`, on successful import calls `transformToAppGraph` and passes result to `<AppGraph>`, shows empty state message for zero-resource graphs ("No resources found in the imported graph"), displays large graph warning when resources > 50 or connections > 100, silently replaces current graph on re-import
- [ ] T014 [US1] Export `PreviewPage` and `previewPageRouteRef` from `plugins/plugin-radius/src/index.ts`
- [ ] T015 [US1] Add `/preview` route to `packages/app/src/App.tsx` — `<Route path="/preview" element={<PreviewPage />} />`
- [ ] T016 [US1] Add "Preview" sidebar item to `packages/app/src/components/Root/Root.tsx` — new `SidebarItem` with appropriate icon (e.g., `Visibility` or `Preview`)
- [ ] T017 [US1] Write unit tests for `PreviewPage` in `plugins/plugin-radius/src/components/preview/PreviewPage.test.tsx` — test: renders import panel, valid import shows graph, invalid import shows errors, empty graph shows message, large graph shows warning, re-import replaces graph

**Checkpoint**: User Story 1 complete — users can navigate to `/preview`, paste or upload JSON, and see the rendered graph. This is the MVP.

---

## Phase 4: User Story 2 — Visual Distinction Between Preview and Live Graphs (Priority: P2)

**Goal**: Preview graphs are visually distinct from live graphs — dashed borders, muted colors, dashed connection lines, and a "Preview" banner — so users never confuse preview with deployed state

**Independent Test**: Compare a preview graph rendering against a live graph side by side; verify dashed node borders, muted colors, dashed connection edges, "Preview" banner, and "Not Deployed" status badges are all present

### Implementation for User Story 2

- [ ] T018 [P] [US2] Modify `AppGraph` component in `packages/rad-components/src/components/appgraph/AppGraph.tsx` — add optional `isPreview?: boolean` prop; when `true`, apply dashed edge styles (not animated), pass `isPreview` flag through node data to `ResourceNode`
- [ ] T019 [P] [US2] Modify `ResourceNode` component in `packages/rad-components/src/components/resourcenode/ResourceNode.tsx` — read `isPreview` from node data; when `true`, render dashed border, muted background color, and "Not Deployed" status badge
- [ ] T020 [P] [US2] Create `PreviewBanner` component in `packages/rad-components/src/components/previewbanner/PreviewBanner.tsx` — prominent banner above graph area showing "Preview" label, distinct styling (color, icon), brief explanatory text (e.g., "This graph was imported from a file and does not represent a deployed application")
- [ ] T021 [P] [US2] Write unit tests for `PreviewBanner` in `packages/rad-components/src/components/previewbanner/PreviewBanner.test.tsx` — test banner renders with correct text and styling
- [ ] T022 [US2] Integrate `PreviewBanner` into `PreviewPage` in `plugins/plugin-radius/src/components/preview/PreviewPage.tsx` — render `<PreviewBanner />` above the graph when data is loaded; pass `isPreview={true}` to `<AppGraph>`
- [ ] T023 [US2] Update `PreviewPage` tests in `plugins/plugin-radius/src/components/preview/PreviewPage.test.tsx` — verify preview banner renders when graph is loaded, verify `isPreview` prop is passed to `AppGraph`

**Checkpoint**: User Stories 1 AND 2 complete — preview graphs are visually distinct from live graphs with all distinction cues present

---

## Phase 5: User Story 3 — Shareable Graph URL (Priority: P3)

**Goal**: Users can generate a shareable URL encoding the preview graph data in the hash fragment; recipients open the URL and see the graph instantly without re-importing

**Independent Test**: Import a graph, generate shareable URL, open URL in a new browser window, verify graph renders identically

### Implementation for User Story 3

- [ ] T024 [P] [US3] Implement `encodeGraphUrl(response, baseUrl): EncodeResult` in `packages/rad-components/src/lib/shareableUrl.ts` — serialize to minified JSON, compress with pako deflate, Base64 URL-safe encode, construct URL with `#graph=` prefix, return error if URL > 64,000 chars per contracts/shareable-url.md
- [ ] T025 [P] [US3] Implement `decodeGraphUrl(hash): ValidationResult<ApplicationGraphResponse>` in `packages/rad-components/src/lib/shareableUrl.ts` — extract `graph=` param from hash, Base64 URL-safe decode, inflate with pako, JSON parse, validate with `validateApplicationGraphResponse`, handle all error cases per contracts/shareable-url.md
- [ ] T026 [P] [US3] Implement `copyShareUrl(response): Promise<EncodeResult>` in `packages/rad-components/src/lib/shareableUrl.ts` — calls `encodeGraphUrl`, updates `window.location.hash`, copies URL to clipboard via `navigator.clipboard.writeText`
- [ ] T027 [P] [US3] Write unit tests for all shareable URL functions in `packages/rad-components/src/lib/shareableUrl.test.ts` — test encode/decode roundtrip, URL length limit error, invalid Base64 decode, corrupted data decode, missing graph param, clipboard mock
- [ ] T028 [US3] Integrate shareable URL into `PreviewPage` in `plugins/plugin-radius/src/components/preview/PreviewPage.tsx` — on mount, check `window.location.hash` for `graph=` param and auto-decode/render; add "Copy Link" / "Share" button that calls `copyShareUrl`; show error message for too-large graphs ("Graph data is too large to share via URL. Export the JSON file instead."); show error for corrupted shared links
- [ ] T029 [US3] Update `PreviewPage` tests in `plugins/plugin-radius/src/components/preview/PreviewPage.test.tsx` — test URL hash decode on mount renders graph, test share button copies URL, test too-large graph shows fallback message, test corrupted URL shows error

**Checkpoint**: User Stories 1, 2, AND 3 complete — shareable URLs work end-to-end

---

## Phase 6: User Story 4 — Export Graph as DOT Format (Priority: P3)

**Goal**: Users can export the current preview graph as a Graphviz DOT file or copy DOT text to clipboard, consistent with the Radius CLI's DOT output format

**Independent Test**: Import a graph, click export, verify downloaded `.dot` file matches expected format and renders correctly with Graphviz

### Implementation for User Story 4

- [ ] T030 [P] [US4] Implement `exportToDot(response, graphName?): string` in `packages/rad-components/src/lib/dotExport.ts` — generate `digraph` with `rankdir=LR`, node styles per contracts/dot-export.md (box/lightblue for Radius resources, ellipse/lightyellow for others), label format `"name\n(type)"`, outbound edges only, deduplicated and sorted, resources sorted by type then name, escape double quotes
- [ ] T031 [P] [US4] Implement `downloadDotFile(dotContent, filename?)` in `packages/rad-components/src/lib/dotExport.ts` — trigger browser download of DOT content as `graph.dot` (default filename) using Blob + anchor click pattern
- [ ] T032 [P] [US4] Implement `copyDotToClipboard(dotContent): Promise<void>` in `packages/rad-components/src/lib/dotExport.ts` — copy DOT string to clipboard via `navigator.clipboard.writeText`
- [ ] T033 [P] [US4] Write unit tests for all DOT export functions in `packages/rad-components/src/lib/dotExport.test.ts` — test DOT output format, node shapes/colors, edge deduplication, sorting, special character escaping, empty graph, download mock, clipboard mock
- [ ] T034 [US4] Create `GraphActions` component in `packages/rad-components/src/components/graphactions/GraphActions.tsx` — action bar with "Export as DOT" button (calls `downloadDotFile`), "Copy DOT" button (calls `copyDotToClipboard`), "Share" / "Copy Link" button (calls `copyShareUrl`); accepts `response: ApplicationGraphResponse` prop; only renders when graph data is present
- [ ] T035 [US4] Write unit tests for `GraphActions` in `packages/rad-components/src/components/graphactions/GraphActions.test.tsx` — test all buttons render, click handlers call correct functions, buttons disabled when no data
- [ ] T036 [US4] Integrate `GraphActions` into `PreviewPage` in `plugins/plugin-radius/src/components/preview/PreviewPage.tsx` — render `<GraphActions>` below the preview banner when graph data is loaded, pass current `ApplicationGraphResponse` as prop; consolidate share button from US3 into `GraphActions`
- [ ] T037 [US4] Update `PreviewPage` tests in `plugins/plugin-radius/src/components/preview/PreviewPage.test.tsx` — verify `GraphActions` renders when graph is loaded, verify export/share actions work through the integrated component

**Checkpoint**: All four user stories complete — full feature set is functional

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: E2E tests, documentation, and final refinements across all user stories

- [ ] T038 [P] Create Playwright E2E test in `packages/app/e2e-tests/preview-page.test.ts` (or project-appropriate E2E location) — test: navigate to `/preview`, paste valid JSON, verify graph renders with preview styling, test share URL generation and navigation, test DOT export button triggers download
- [ ] T039 [P] Update component barrel exports in `packages/rad-components/src/index.ts` to export new components (`GraphImportPanel`, `PreviewBanner`, `GraphActions`) and utility functions (`parseGraphJson`, `transformToAppGraph`, `validateApplicationGraphResponse`, `exportToDot`, `encodeGraphUrl`, `decodeGraphUrl`)
- [ ] T040 [P] Add Storybook stories for `PreviewBanner` in `packages/rad-components/src/components/previewbanner/__docs__/PreviewBanner.stories.tsx` and `GraphActions` in `packages/rad-components/src/components/graphactions/__docs__/GraphActions.stories.tsx`
- [ ] T041 Run quickstart.md validation — verify all high-level changes listed in quickstart.md are implemented and functional
- [ ] T042 Code cleanup — verify all imports are used, remove any TODO comments, ensure consistent error message formatting across validation/decode/export modules

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup (T001/T002) — BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational (Phase 2) completion
- **User Story 2 (Phase 4)**: Depends on Foundational (Phase 2) completion; independent of US1 for component work (T018-T021), but integration (T022-T023) requires PreviewPage from US1
- **User Story 3 (Phase 5)**: Depends on Foundational (Phase 2) completion; lib work (T024-T027) is independent, but integration (T028-T029) requires PreviewPage from US1
- **User Story 4 (Phase 6)**: Depends on Foundational (Phase 2) completion; lib work (T030-T033) is independent, but integration (T034-T037) requires PreviewPage from US1 and shareableUrl from US3
- **Polish (Phase 7)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational — no dependencies on other stories
- **User Story 2 (P2)**: Component creation (T018-T021) can start after Foundational in parallel with US1; integration (T022-T023) requires US1 PreviewPage
- **User Story 3 (P3)**: Library functions (T024-T027) can start after Foundational in parallel with US1; integration (T028-T029) requires US1 PreviewPage
- **User Story 4 (P3)**: Library/component (T030-T035) can start after Foundational in parallel; integration (T036-T037) requires US1 PreviewPage and US3 share button

### Within Each User Story

- Library/utility modules before UI components
- UI components before page integration
- Page integration before page-level tests

### Parallel Opportunities per Story

**User Story 1**: T008, T009, T010 can run in parallel (different files)
**User Story 2**: T018, T019, T020, T021 can run in parallel (different files)
**User Story 3**: T024, T025, T026, T027 can run in parallel (different files)
**User Story 4**: T030, T031, T032, T033 can run in parallel (different files)
**Cross-story**: US2 components (T018-T021), US3 lib (T024-T027), and US4 lib (T030-T033) can all run in parallel with US1 implementation

---

## Parallel Example: User Story 1

```bash
# After Foundational phase completes, launch these in parallel:
Task T008: "Create GraphImportPanel component in packages/rad-components/src/components/graphimport/GraphImportPanel.tsx"
Task T009: "Write unit tests for GraphImportPanel in packages/rad-components/src/components/graphimport/GraphImportPanel.test.tsx"
Task T010: "Create Storybook stories for GraphImportPanel in packages/rad-components/src/components/graphimport/__docs__/GraphImportPanel.stories.tsx"

# Then sequentially:
Task T011: "Add previewPageRouteRef to plugins/plugin-radius/src/routes.ts"
Task T012: "Add PreviewPage routable extension to plugins/plugin-radius/src/plugin.ts"
Task T013: "Create PreviewPage component" (depends on T008, T011, T012)
Task T014: "Export from index.ts"
Task T015: "Add route to App.tsx"
Task T016: "Add sidebar item to Root.tsx"
Task T017: "Write PreviewPage tests" (depends on T013)
```

---

## Parallel Example: Cross-Story Library Work

```bash
# These library tasks can ALL run in parallel after Foundational phase:
Task T024: "Implement encodeGraphUrl in packages/rad-components/src/lib/shareableUrl.ts"  (US3)
Task T030: "Implement exportToDot in packages/rad-components/src/lib/dotExport.ts"        (US4)
Task T018: "Modify AppGraph for preview styling"                                            (US2)
Task T020: "Create PreviewBanner component"                                                 (US2)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T002)
2. Complete Phase 2: Foundational (T003-T007) — CRITICAL, blocks all stories
3. Complete Phase 3: User Story 1 (T008-T017)
4. **STOP and VALIDATE**: Navigate to `/preview`, paste sample JSON, verify graph renders
5. Deploy/demo if ready — users can already visualize graphs

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Preview graphs are visually distinct → Deploy/Demo
4. Add User Story 3 → Shareable URLs work → Deploy/Demo
5. Add User Story 4 → DOT export works → Deploy/Demo
6. Polish → E2E tests, Storybook, cleanup → Final release

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - **Developer A**: User Story 1 (PreviewPage, routing, import panel)
   - **Developer B**: User Story 2 components (AppGraph preview styles, PreviewBanner) + User Story 3 lib (shareableUrl.ts)
   - **Developer C**: User Story 4 lib (dotExport.ts) + GraphActions component
3. Integration tasks (T022, T028, T036) happen sequentially after US1 PreviewPage is ready

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [Story] label maps each task to its user story for traceability
- Each user story should be independently completable and testable once its dependencies are met
- Commit after each task or logical group
- Stop at any checkpoint to validate the story independently
- The `dashboard/` repository uses Yarn 4 workspaces — run `yarn install` from repo root
- Use `backstage-cli` for running tests: `yarn workspace @backstage/plugin-radius test`
- All new TypeScript files must follow strict mode and existing ESLint/Prettier config
