# Tasks: `rad app graph --file` Static Graph from Bicep

**Input**: Design documents from `/specs/002-app-graph-file/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Single project**: Go monorepo at `radius/` repository
- Source: `pkg/cli/cmd/app/graph/` (CLI command), `pkg/corerp/frontend/controller/applications/` (graph computation)
- Tests: Co-located `*_test.go` files + `testdata/` fixtures

---

## Phase 1: Setup

**Purpose**: Export shared function and create test fixture infrastructure

- [X] T001 Export `computeGraph` → `ComputeGraph` in `pkg/corerp/frontend/controller/applications/graph_util.go` and update all internal callers in the same package to use the exported name
- [X] T002 [P] Create ARM JSON test fixture files in `pkg/cli/cmd/app/graph/testdata/` per quickstart.md fixture table: `simple-app.json` (1 app, 2 containers, 1 connection), `no-app.json` (resources without application resource), `multi-app.json` (2 application resources), `non-radius.json` (mixed Radius and Azure resources), `unresolvable.json` (parameterized connection sources), `empty.json` (empty resources map), `with-modules.json` (Microsoft.Resources/deployments entries)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Template extraction helper functions that ALL user stories depend on

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [X] T003 Create `pkg/cli/cmd/app/graph/template.go` with `resourceEntry` struct, `stripAPIVersion`, `synthesizeResourceID`, `isRadiusResource`, and `resolveExpression` functions per contracts/template-extraction.md
- [X] T004 Create `pkg/cli/cmd/app/graph/template_test.go` with unit tests for `stripAPIVersion` (type with/without API version), `synthesizeResourceID` (valid ID format), `isRadiusResource` (Radius import, Applications.* prefix, Radius.* prefix, non-Radius), and `resolveExpression` (literal pass-through, `[reference('X').id]` match, unsupported expression returns false)

**Checkpoint**: All helper functions tested and working — extractResourcesFromTemplate can now be built on top

---

## Phase 3: User Story 1 — Preview Application Graph from Bicep File (Priority: P1) 🎯 MVP

**Goal**: Run `rad app graph --file app.bicep` and see the resource topology and connections offline

**Independent Test**: Create a Bicep file with interconnected Radius resources, run `rad app graph --file <path>`, verify text output shows resources and connections in the same format as live `rad app graph`

### Implementation for User Story 1

- [X] T005 [US1] Implement `extractResourcesFromTemplate` in `pkg/cli/cmd/app/graph/template.go` — iterate ARM JSON resources map, build `resourceEntry` lookup, resolve expressions in connections/routes/application fields, construct `[]generated.GenericResource` with synthesized IDs, detect conditions/modules/non-Radius resources for warnings, sort deterministically
- [X] T006 [US1] Implement `scopeToApplication` in `pkg/cli/cmd/app/graph/template.go` — count application resources from `[]generated.GenericResource` by checking `Type` field (0 → all resources implicit app, 1 → filter to referencing resources, 2+ → error), return app name and filtered resource list. Note: this replaces the `countApplicationResources` helper from the contract, which operates on the internal `resourceEntry` type; `scopeToApplication` counts directly from the post-extraction `GenericResource` slice, making `countApplicationResources` unnecessary as a separate exported function
- [X] T007 [US1] Add unit tests for `extractResourcesFromTemplate` and `scopeToApplication` in `pkg/cli/cmd/app/graph/template_test.go` — test with simple-app fixture (correct GenericResource construction, synthesized IDs, resolved connections), empty resources (no error, empty list), non-Radius resources (included with minimal properties), scopeToApplication with 1 app (filtered result)
- [X] T008 [P] [US1] Add `FilePath string` and `BicepClient bicep.Interface` fields to `Runner` struct, register `--file` / `-f` string flag in `NewCommand()` in `pkg/cli/cmd/app/graph/graph.go`
- [X] T009 [US1] Implement file-mode branch in `Validate()` in `pkg/cli/cmd/app/graph/graph.go` — read `--file` flag, check mutual exclusivity with positional app name arg, validate file exists with `os.Stat`, read `--output` flag, skip workspace/scope/application validation
- [X] T010 [US1] Implement file-mode branch in `Run()` in `pkg/cli/cmd/app/graph/graph.go` — call `PrepareTemplate(FilePath)`, call `extractResourcesFromTemplate`, emit warnings to stderr, call `scopeToApplication`, call `applications.ComputeGraph(appResources, nil)`, switch on format for text (`display()`), JSON (`Output.WriteFormatted`), or DOT (`displayDot()`) output
- [X] T011 [US1] Add integration tests for `--file` mode in `pkg/cli/cmd/app/graph/graph_test.go` — test with `simple-app.json` fixture producing expected text output, test with `--output json` producing valid `ApplicationGraphResponse` JSON, test with `--output dot` producing valid DOT digraph output, test with `empty.json` producing empty graph message, test file-not-found path producing error

**Checkpoint**: `rad app graph --file <path>` works end-to-end with text and JSON output — MVP is functional

---

## Phase 4: User Story 2 — Mutual Exclusivity of `--file` and App Name (Priority: P1)

**Goal**: Clear error when both `--file` and positional app name are provided

**Independent Test**: Run `rad app graph myapp --file app.bicep` and verify the error message

**Note**: Implementation is in T009 (Validate branch includes the mutual exclusivity check). This phase adds dedicated tests verifying US2 acceptance scenarios.

- [ ] T012 [US2] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: `--file` + positional app name → error message `"--file and application name are mutually exclusive"`
- [ ] T013 [P] [US2] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: positional app name only (no `--file`) → existing live-mode validation path executes unchanged

**Checkpoint**: Mutual exclusivity validated — both conflict and non-conflict paths tested

---

## Phase 5: User Story 3 — Graceful Handling of Unresolvable Connections (Priority: P2)

**Goal**: Partial graph with warnings when connections use parameterized or expression-based sources

**Independent Test**: Create a Bicep file with `source: someParam` connection, run `rad app graph --file <path>`, verify graph shows resolvable resources and stderr shows warning

**Note**: Core warning mechanism is in `resolveExpression` (T003) and `extractResourcesFromTemplate` (T005). This phase adds dedicated tests for degradation scenarios.

- [ ] T014 [US3] Add unit tests for unresolvable expression handling in `pkg/cli/cmd/app/graph/template_test.go` — `[parameters('X')]` → warning + connection skipped, `[format(...)]` → warning + connection skipped, partially resolvable template (some connections resolve, some don't) → partial results + warnings list
- [ ] T015 [P] [US3] Add unit tests for conditional resource and module reference warnings in `pkg/cli/cmd/app/graph/template_test.go` — resource with `condition` field → included in output + warning, resource with type `Microsoft.Resources/deployments` → warning about module not traversed
- [ ] T016 [US3] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: `--file` with `unresolvable.json` fixture → partial graph rendered + stderr contains warning messages

**Checkpoint**: Graceful degradation verified — unresolvable connections warn, don't break

---

## Phase 6: User Story 4 — Offline Operation Without Radius Environment (Priority: P2)

**Goal**: `rad app graph --file` works with no Radius control plane, no workspace configured

**Independent Test**: Run the command with no Radius environment configured, verify success with no network calls

**Note**: Offline operation is inherent in the design — `--file` mode never calls Radius APIs. This phase adds tests confirming the offline guarantee and JSON output for CI pipelines.

- [ ] T017 [US4] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: `--file` mode with no workspace or environment configured → command succeeds without error (verifies Validate skips workspace resolution)
- [ ] T018 [P] [US4] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: `--file` + `--output json` → valid JSON output conforming to `ApplicationGraphResponse` schema with `provisioningState: "NotDeployed"` and empty `outputResources` on all resources

**Checkpoint**: Offline and CI-pipeline use cases validated

---

## Phase 7: User Story 5 — Handling Files with Multiple or No Application Resources (Priority: P3)

**Goal**: Sensible behavior for 0, 1, or multiple application resources in a Bicep file

**Independent Test**: Create Bicep files with 0 and 2 application resources, verify implicit-app and error behaviors respectively

- [ ] T019 [US5] Add unit tests for `scopeToApplication` edge cases in `pkg/cli/cmd/app/graph/template_test.go` — 0 apps → returns all resources with empty app name, 1 app → returns filtered resources with app name, 2 apps → returns error mentioning multiple applications
- [ ] T020 [US5] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: `--file` with `no-app.json` → all resources displayed in implicit application graph
- [ ] T021 [P] [US5] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: `--file` with `multi-app.json` → error message indicating multiple applications found

**Checkpoint**: All application scoping behaviors verified — 0/1/many cases handled correctly

---

## Phase 8: User Story 6 — Visual Graph Output via Graphviz DOT Format (Priority: P2)

**Goal**: Generate Graphviz DOT output that can be rendered to PNG/SVG diagrams

**Independent Test**: Run `rad app graph --file app.bicep --output dot | dot -Tpng -o graph.png` and verify a valid PNG is produced

**Note**: The `--output dot` format switch is wired in T010 (Run branch) for file mode. This phase implements the `displayDot()` function, its tests, and adds `--output dot` support to the existing live-mode code path (per FR-021 and US6 Scenario 4).

- [ ] T022 [US6] Create `pkg/cli/cmd/app/graph/display_dot.go` with `displayDot(resources []*ApplicationGraphResource, appName string) string` — produce valid Graphviz DOT digraph with `rankdir=LR`, Radius resource nodes as boxes (lightblue fill, label `name\n(type)`), non-Radius resource nodes as ellipses (lightyellow fill), directed edges for outbound connections, deduplicated edges, deterministic ordering
- [ ] T023 [US6] Create `pkg/cli/cmd/app/graph/display_dot_test.go` with unit tests — single resource (valid digraph wrapper + one node), two resources with connection (node + directed edge), non-Radius resource (ellipse shape, lightyellow), empty resources (empty digraph), deterministic output (same input → same output), special characters in names are escaped
- [ ] T024 [US6] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: `--file` with `simple-app.json` + `--output dot` → output starts with `digraph`, contains node labels matching resource names and types, contains edge `->` for connections
- [ ] T025 [P] [US6] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: `--file` with `non-radius.json` + `--output dot` → non-Radius resources use `shape=ellipse` and `fillcolor=lightyellow`
- [ ] T026 [US6] Modify live-mode `Run()` branch in `pkg/cli/cmd/app/graph/graph.go` to handle `--output dot` — after `computeGraph()` returns in the existing live path, add format switch case for `"dot"` calling `displayDot(response.Resources, appName)` so that `rad app graph myapp --output dot` works without `--file`
- [ ] T027 [US6] Add integration test in `pkg/cli/cmd/app/graph/graph_test.go`: live-mode `rad app graph myapp --output dot` → output starts with `digraph`, contains expected node labels and edges (requires mock API client returning test resources)

**Checkpoint**: `--output dot` produces valid Graphviz DOT in both `--file` mode and live mode — visual graph generation works

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, code quality, and end-to-end validation

- [ ] T028 [P] Add godoc comments to all exported functions and types in `pkg/cli/cmd/app/graph/template.go` and `pkg/cli/cmd/app/graph/display_dot.go`
- [ ] T029 [P] Update `rad app graph` reference documentation in docs repo to document the `--file` flag, `--output dot` format, file mode behavior, and example usage including `--output dot | dot -Tpng -o graph.png`
- [ ] T030 Run quickstart.md success verification checklist end-to-end (11 verification items)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 (T001 for `ComputeGraph` export) — BLOCKS all user stories
- **US1 (Phase 3)**: Depends on Phase 2 completion — core feature
- **US2 (Phase 4)**: Depends on Phase 3 (implementation in T009)
- **US3 (Phase 5)**: Depends on Phase 3 (implementation in T005)
- **US4 (Phase 6)**: Depends on Phase 3 (implementation in T009-T010)
- **US5 (Phase 7)**: Depends on Phase 3 (implementation in T006)
- **US6 (Phase 8)**: Depends on Phase 3 (T010 wires the file-mode format switch); `displayDot()` implementation is independent; T026 (live-mode DOT) depends on T022 (`displayDot` exists)
- **Polish (Phase 9)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Can start after Foundational (Phase 2) — no dependencies on other stories
- **US2 (P1)**: Implementation is embedded in US1 (T009); tests can run after US1 is complete
- **US3 (P2)**: Implementation is embedded in US1 (T003, T005); tests can run after US1 is complete
- **US4 (P2)**: Implementation is inherent in design; tests can run after US1 is complete
- **US5 (P3)**: Implementation is in US1 (T006); edge case tests can run after US1 is complete
- **US6 (P2)**: `displayDot()` is a new function (T022); file-mode format switch wired in US1 (T010); live-mode DOT added in T026; tests can run after T022 + T010 are complete

### Within Phase 3 (US1)

- T005 → T006 (both template.go, sequential)
- T007 depends on T005, T006 (tests their functions)
- T008 is [P] with T005-T007 (different file: graph.go)
- T009 depends on T008 (same file: graph.go)
- T010 depends on T009 (same file: graph.go)
- T011 depends on T010 and T005-T007 (integration tests need all implementation)

### Parallel Opportunities

- **Phase 1**: T001 and T002 can run in parallel (different directories)
- **Phase 3**: T008 (graph.go changes) can run in parallel with T005-T007 (template.go/template_test.go)
- **Phase 4**: T012 and T013 can run in parallel (independent test functions)
- **Phase 5**: T014 and T015 can run in parallel (independent test functions)
- **Phase 6**: T017 and T018 can run in parallel (independent test functions)
- **Phase 7**: T020 and T021 can run in parallel (independent test functions)
- **Phase 8**: T022 (display_dot.go) can run in parallel with T005-T007 (different file); T024 and T025 can run in parallel (independent test functions); T026 depends on T022; T027 depends on T026
- **Phase 9**: T028 and T029 can run in parallel (different repos)
- **Phases 4-8**: Once US1 is complete, Phases 4-8 can ALL proceed in parallel (they add code/tests to different functions/files)

---

## Parallel Example: User Story 1

```text
# Stream A: template.go + template_test.go
Task T005: Implement extractResourcesFromTemplate in template.go
Task T006: Implement scopeToApplication in template.go
Task T007: Unit tests in template_test.go

# Stream B: graph.go (can run in parallel with Stream A)
Task T008: Add Runner fields, register --file flag in graph.go

# Sequential after both streams:
Task T009: Validate() branch in graph.go (after T008)
Task T010: Run() branch in graph.go (after T009)
Task T011: Integration tests in graph_test.go (after T010 + T007)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (export ComputeGraph, create fixtures)
2. Complete Phase 2: Foundational (helper functions + tests)
3. Complete Phase 3: User Story 1 (core feature)
4. **STOP and VALIDATE**: Run `rad app graph --file testdata/simple-app.json` and verify output
5. Demo if ready — the core feature is usable

### Incremental Delivery

1. Setup + Foundational → Helper functions ready
2. Add US1 → Test independently → Core feature works (MVP!)
3. Add US2 tests → Verify mutual exclusivity
4. Add US3 tests → Verify graceful degradation
5. Add US4 tests → Verify offline guarantee
6. Add US5 tests → Verify application scoping edge cases
7. Add US6 → Implement `displayDot()` + live-mode DOT + tests → Visual graph output works in both modes
8. Polish → Documentation, godoc, final validation
9. Each story's tests add confidence without breaking previous stories

### Key Files Summary

| File | Action | Phases |
|------|--------|--------|
| `pkg/corerp/frontend/controller/applications/graph_util.go` | Modify (export) | Phase 1 |
| `pkg/cli/cmd/app/graph/testdata/*.json` | Create (7 files) | Phase 1 |
| `pkg/cli/cmd/app/graph/template.go` | Create | Phase 2-3 |
| `pkg/cli/cmd/app/graph/template_test.go` | Create | Phase 2-3, 5, 7 |
| `pkg/cli/cmd/app/graph/graph.go` | Modify | Phase 3, 8 |
| `pkg/cli/cmd/app/graph/graph_test.go` | Modify | Phase 3-8 |
| `pkg/cli/cmd/app/graph/display_dot.go` | Create | Phase 8 |
| `pkg/cli/cmd/app/graph/display_dot_test.go` | Create | Phase 8 |

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [Story] label maps task to specific user story for traceability
- US2/US3/US4 phases are primarily test phases — their implementation is embedded in US1's code changes
- US6 adds new code (`display_dot.go`) and modifies `graph.go` for both file-mode (T010) and live-mode (T026) DOT output
- Commit after each task or logical group
- Stop at any checkpoint to validate the story independently
- Reference contracts/ directory for exact function signatures and behavior contracts
- Reference quickstart.md for test fixture content and success verification checklist
