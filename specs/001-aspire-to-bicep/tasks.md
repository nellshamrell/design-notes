# Tasks: Aspire Manifest to Bicep Conversion

**Input**: Design documents from `/specs/001-aspire-to-bicep/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅, quickstart.md ✅

**Tests**: Included — plan.md specifies unit tests (parser, mapper, emitter/golden file) as a core part of the testing strategy.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- Include exact file paths in descriptions

## Path Conventions

All source paths are relative to the `radius` repository root (`/home/nell/projects/radius/`):

- **New package**: `pkg/cli/cmd/aspire/` (parent command) and `pkg/cli/cmd/aspire/convert/` (subcommand)
- **Test data**: `pkg/cli/cmd/aspire/convert/testdata/`
- **Modified file**: `cmd/rad/cmd/root.go` (wire new command)

---

## Phase 1: Setup

**Purpose**: Create the package structure and wire the new `rad aspire` command group into the existing CLI.

- [X] T001 Create directory structure `pkg/cli/cmd/aspire/convert/testdata/` in the radius repository
- [X] T002 Create parent cobra command `rad aspire` with Use, Short, Long, and Example fields in `pkg/cli/cmd/aspire/aspire.go`
- [X] T003 Wire `aspireCmd` into the root command's `initSubCommands()` in `cmd/rad/cmd/root.go`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Define all Go types (Aspire parse-side and Bicep IR), implement the manifest parser, implement the Bicep text emitter framework, and create the convert command skeleton. These are required by ALL user stories.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T004 Define Aspire manifest Go types (AspireManifest, AspireResource, AspireBinding, AspireBuild, AspireInput, AspireInputDefault, AspireGenerate) with JSON struct tags in `pkg/cli/cmd/aspire/convert/manifest.go`
- [ ] T005 Implement Aspire manifest JSON parser function (`Parse`) that deserializes JSON into AspireManifest, populates resource Name fields from map keys, and validates required fields in `pkg/cli/cmd/aspire/convert/manifest.go`
- [ ] T006 [P] Define Bicep IR Go types (BicepFile, BicepParameter, BicepResource, BicepContainer, BicepPort, BicepEnvVar, BicepConnection, BicepGateway, BicepGatewayRoute, BicepComment) in `pkg/cli/cmd/aspire/convert/emitter.go`
- [ ] T007 [P] Implement Bicep text emitter (`Emit` function) using Go `text/template` with templates for extension declarations, parameters, application resource, containers, data stores, gateways, and unsupported-resource comments in `pkg/cli/cmd/aspire/convert/emitter.go`
- [ ] T008 [P] Copy sample `aspire-manifest.json` from the repository root to `pkg/cli/cmd/aspire/convert/testdata/aspire-manifest.json`
- [ ] T009 Create convert command skeleton with `NewCommand` (Cobra + flags), `Runner` struct, `Validate` (check manifest arg exists), and `Run` (orchestrate Parse → Map → Emit → Write) in `pkg/cli/cmd/aspire/convert/convert.go`

**Checkpoint**: Foundation ready — all types defined, parser and emitter implemented, command skeleton in place. User story implementation can now begin.

---

## Phase 3: User Story 1 — Convert a Basic Aspire Manifest (Priority: P1) 🎯 MVP

**Goal**: Run `rad aspire convert aspire-manifest.json` and get a valid `app.bicep` with application, container resources (ports, env vars), inter-resource connections, backing-service resources, and gateway resources for external bindings.

**Independent Test**: Convert the sample manifest → verify output Bicep contains expected application, container, data-store, gateway, and connection resources. Golden file comparison validates structure and content. Output compiles with Radius Bicep toolchain.

### Implementation for User Story 1

- [ ] T010 [US1] Implement extensible resource type mapping table (Aspire type string → Radius resource type + category) in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T011 [US1] Implement container mapping for `container.v0` and `container.v1` → `BicepContainer` with image, command/args, and `ApplicationRef`/`EnvironmentRef` wiring in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T012 [US1] Implement Aspire binding → `BicepPort` mapping with scheme/protocol/targetPort translation in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T013 [US1] Implement regex-based expression reference parser that extracts `{resource.property.path}` patterns and resolves them to Bicep resource references or parameter references in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T014 [US1] Implement connection generation: detect `connectionString` and binding references across resources and produce `BicepConnection` entries on consuming containers in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T015 [US1] Implement backing-service mapping (redis.server.v0, postgres.server.v0, mysql.server.v0 → Radius data-store `BicepResource`) with mapping table entries per FR-016 in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T016 [US1] Implement gateway/route generation for containers with `external: true` bindings → `BicepGateway` with routes per FR-017 in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T017 [US1] Implement top-level `MapManifest` function orchestrating: extension collection, application resource, environment parameter, iterate resources by type → delegate to container/backing-service/gateway sub-mappers in `pkg/cli/cmd/aspire/convert/mapper.go`

### Tests for User Story 1

- [ ] T018 [P] [US1] Create `expected-basic.bicep` golden file for basic container conversion test in `pkg/cli/cmd/aspire/convert/testdata/expected-basic.bicep`
- [ ] T019 [P] [US1] Write parser unit tests with table-driven cases covering: valid container, valid backing service, missing fields, unknown type, malformed JSON, empty resources in `pkg/cli/cmd/aspire/convert/manifest_test.go`
- [ ] T020 [P] [US1] Write mapper unit tests with table-driven cases for: container mapping, binding→port, expression resolution, connection generation, backing-service mapping, gateway generation, extension collection in `pkg/cli/cmd/aspire/convert/mapper_test.go`
- [ ] T021 [P] [US1] Write emitter golden file test comparing full Emit output against `expected-basic.bicep` in `pkg/cli/cmd/aspire/convert/emitter_test.go`

**Checkpoint**: User Story 1 is fully functional. `rad aspire convert` produces a valid Bicep file for manifests with containers, backing services, connections, and gateways. All tests pass.

---

## Phase 4: User Story 2 — Handle Parameters and Secrets (Priority: P2)

**Goal**: Aspire `parameter.v0` resources with `secret: true` inputs are converted to `@secure()` Bicep parameters, and container environment variables referencing parameter values are correctly wired.

**Independent Test**: Convert a manifest containing `parameter.v0` resources with secret inputs → verify output Bicep declares `@secure()` parameters and env vars reference those parameters (no inline secrets). Golden file validates output.

### Implementation for User Story 2

- [ ] T022 [US2] Implement `parameter.v0` resource mapping: detect `secret` flag on inputs, generate `BicepParameter` with `Secure: true` and description in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T023 [US2] Implement parameter value wiring: resolve `{param.value}` and `{param.inputs.name}` expression patterns in container env vars to Bicep parameter references in `pkg/cli/cmd/aspire/convert/mapper.go`

### Tests for User Story 2

- [ ] T024 [P] [US2] Create `expected-secrets.bicep` golden file for secure parameter conversion in `pkg/cli/cmd/aspire/convert/testdata/expected-secrets.bicep`
- [ ] T025 [P] [US2] Add parameter mapping test cases (secret vs non-secret, value wiring, missing inputs) to `pkg/cli/cmd/aspire/convert/mapper_test.go`
- [ ] T026 [P] [US2] Add secrets golden file comparison test to `pkg/cli/cmd/aspire/convert/emitter_test.go`

**Checkpoint**: User Stories 1 AND 2 both work. Manifests with containers + parameters + secrets produce correct, secure Bicep output.

---

## Phase 5: User Story 3 — Specify Output Path and Overwrite Behavior (Priority: P3)

**Goal**: Users control output file path with `--output`, prevent accidental overwrites (default), and allow forced overwrites with `--force`. Custom application name via `--application`.

**Independent Test**: Run command with `--output custom.bicep` → file appears at custom path. Run when file exists without `--force` → error. Run with `--force` → file overwritten. Run with `--application my-app` → app resource has custom name.

### Implementation for User Story 3

- [ ] T027 [US3] Register `--output` (`-o`, default `app.bicep`), `--force` (`-f`), and `--application` (`-a`) flags on the Cobra command in `pkg/cli/cmd/aspire/convert/convert.go`
- [ ] T028 [US3] Implement `Validate` method: check input file exists via `filesystem.FileSystem`, check output file existence, error if exists and `--force` not set, derive application name from flag or manifest in `pkg/cli/cmd/aspire/convert/convert.go`
- [ ] T029 [US3] Implement file write in `Run` using `filesystem.FileSystem` to write emitted Bicep string to the resolved output path in `pkg/cli/cmd/aspire/convert/convert.go`
- [ ] T030 [US3] Write command validation and flag handling tests: missing input, output path default, custom output, overwrite blocked, force overwrite, custom application name in `pkg/cli/cmd/aspire/convert/convert_test.go`

**Checkpoint**: User Story 3 complete. Output control and safety features work as expected.

---

## Phase 6: User Story 4 — Report Unsupported Aspire Resource Types (Priority: P3)

**Goal**: Unrecognized Aspire resource types produce clear warnings on stderr, comments in the Bicep output, and the conversion summary lists all skipped resources. `container.v1` build configs produce specific advisory warnings.

**Independent Test**: Add an unsupported resource type to a manifest → convert → verify warning printed, Bicep comment present, conversion still succeeds. Convert manifest with all supported types → verify no warnings.

### Implementation for User Story 4

- [ ] T031 [US4] Implement unsupported resource type detection: resources not in the mapping table generate `BicepComment` entries and append to `BicepFile.Warnings` in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T032 [US4] Implement `container.v1` build configuration warning per FR-014: detect `Build` field, set `NeedsBuildWarning` on `BicepContainer`, append advisory warning in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T033 [US4] Implement conversion summary output in `Run`: print converted resource list, warnings, and generated file stats (container count, parameter count, gateway count, skipped count) to stdout in `pkg/cli/cmd/aspire/convert/convert.go`
- [ ] T034 [US4] Add unsupported resource and build-warning test cases to `pkg/cli/cmd/aspire/convert/mapper_test.go`

**Checkpoint**: User Story 4 complete. All unsupported resources produce actionable warnings.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Edge case handling, full golden file validation, and end-to-end quickstart verification.

- [ ] T035 [P] Handle edge cases in mapper: empty manifest (app-only output with warning), dangling references (warn and skip connection), name collisions (disambiguate with suffix), unknown schema version (warn and attempt best-effort) in `pkg/cli/cmd/aspire/convert/mapper.go`
- [ ] T036 [P] Handle edge cases in command: invalid JSON error message, missing file error message, unreadable file error with descriptive exit code 1 in `pkg/cli/cmd/aspire/convert/convert.go`
- [ ] T037 Create `expected-full.bicep` golden file for complete sample manifest conversion (containers + secrets + gateways + data stores + unsupported comments) in `pkg/cli/cmd/aspire/convert/testdata/expected-full.bicep`
- [ ] T038 Write end-to-end emitter golden file test for full manifest scenario comparing against `expected-full.bicep` in `pkg/cli/cmd/aspire/convert/emitter_test.go`
- [ ] T039 Run quickstart.md validation: execute full conversion pipeline against sample manifest, verify output compiles, review summary output matches quickstart expectations

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories
- **User Stories (Phase 3–6)**: All depend on Foundational phase completion
  - US1 (Phase 3): Must complete before US2 (Phase 4) since parameter wiring extends mapper logic
  - US2 (Phase 4): Depends on US1 mapper infrastructure
  - US3 (Phase 5): Can run in parallel with US1/US2 (different file: convert.go vs mapper.go)
  - US4 (Phase 6): Can run in parallel with US2/US3 (extends mapper.go but independent logic paths)
- **Polish (Phase 7)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational — no story dependencies. **This is the MVP.**
- **User Story 2 (P2)**: Depends on US1 mapper infrastructure (expression resolver, MapManifest orchestration)
- **User Story 3 (P3)**: Independent of other stories — modifies convert.go (command layer), not mapper.go
- **User Story 4 (P3)**: Depends on US1 mapper infrastructure (mapping table, resource iteration loop)

### Within Each User Story

- Implementation tasks before test tasks (tests validate the implementation)
- Mapping table before specific mappers (T010 before T011–T017)
- Expression resolver (T013) before connection generation (T014) and parameter wiring (T023)
- Top-level orchestrator (T017) after all sub-mappers
- Golden files can be written in parallel with mapper tests

### Parallel Opportunities

**Phase 2 (Foundational)**:
```
T004, T005 (manifest.go — sequential, types then parser)
T006 ─────── (emitter.go — parallel with T004/T005)
T007 ─────── (emitter.go — after T006)
T008 ─────── (testdata — parallel with everything)
T009 ─────── (convert.go — after T006/T007 for type imports)
```

**Phase 3 (US1) — Implementation then tests**:
```
T010 → T011 → T012 → T013 → T014 (sequential mapper build-up)
T015 ──────────────────────────── (parallel with T011–T014, different resource category)
T016 ──────────────────────────── (parallel with T011–T014, different resource category)
T017 ─────────────────────────────── (after T011–T016, orchestrates all)

T018 ── (golden file — parallel with T019–T021)
T019 ── (manifest_test.go — parallel)
T020 ── (mapper_test.go — parallel with T019)
T021 ── (emitter_test.go — parallel with T019–T020)
```

**Phase 4 (US2)**:
```
T022 → T023 (sequential — mapping then wiring)
T024 ── (golden file — parallel with T025–T026)
T025 ── (mapper_test.go — parallel)
T026 ── (emitter_test.go — parallel)
```

**Phase 5 (US3)**: T027 → T028 → T029 → T030 (all in convert.go, sequential)

**Phase 6 (US4)**: T031, T032 can be parallel (different concerns in mapper.go); T033 after both; T034 after T031–T032.

**Phase 7 (Polish)**: T035, T036 in parallel (different files); T037 → T038 sequential (golden file then test); T039 last.

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001–T003)
2. Complete Phase 2: Foundational (T004–T009) — **CRITICAL, blocks all stories**
3. Complete Phase 3: User Story 1 (T010–T021)
4. **STOP and VALIDATE**: Run `go test ./pkg/cli/cmd/aspire/...` — all tests pass, golden file matches
5. Run `rad aspire convert aspire-manifest.json` — produces valid `app.bicep`
6. The MVP is deployable with `rad deploy`

### Incremental Delivery

1. **Setup + Foundational** → Package and types ready
2. **Add US1** → Core conversion works → Test + validate → **MVP!**
3. **Add US2** → Secrets handled → Test golden file → Deploy with `--parameters`
4. **Add US3** → Output control + safety → Test flags
5. **Add US4** → Warnings + summary → Test with unsupported resources
6. **Polish** → Edge cases, full golden file, quickstart validation
7. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1 (mapper.go — core logic)
   - Developer B: User Story 3 (convert.go — command layer, independent of mapper)
3. After US1 completes:
   - Developer A: User Story 2 (extends mapper)
   - Developer B: User Story 4 (extends mapper, independent paths)
4. All: Polish phase

---

## Notes

- [P] tasks = different files, no dependencies on incomplete tasks
- [Story] label maps task to specific user story for traceability
- All paths are relative to the `radius` repository root
- The `--output` flag naming may need adjustment if it conflicts with the inherited Cobra output-format flag — resolve during T027 implementation (contract notes `--out-file` as fallback)
- Backing-service mapping uses new-style `Radius.*` resource types per research.md decision; the mapping table (T010) must be easily updatable
- Golden files are the primary validation mechanism — keep them updated as mapper logic evolves
- Commit after each task or logical group; stop at any checkpoint to validate independently
