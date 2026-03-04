# Implementation Plan: `rad app graph --file` Static Graph from Bicep

**Branch**: `002-app-graph-file` | **Date**: 2026-03-03 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/002-app-graph-file/spec.md`

## Summary

Add a `--file` flag to the `rad app graph` CLI command that compiles a Bicep or ARM JSON file locally and produces the same application graph output without requiring a deployed Radius environment. The implementation reuses the existing `PrepareTemplate()` for compilation, builds `[]generated.GenericResource` objects from the ARM JSON template, and feeds them into the existing `computeGraph()` function to produce the graph. Connection sources are resolved via ARM reference expression parsing and the existing `findSourceResource()` hostname matching. Non-Radius resources are included as untyped graph nodes.

## Technical Context

**Language/Version**: Go 1.26.0
**Primary Dependencies**: Cobra CLI framework (`github.com/spf13/cobra v1.10.2`), testify (`github.com/stretchr/testify`), gomock (`go.uber.org/mock`)
**Storage**: N/A (no persistence; reads from filesystem only)
**Testing**: Standard `testing` package + testify/require + gomock; shared CLI test helpers via `radcli.SharedCommandValidation` and `radcli.SharedValidateValidation`
**Target Platform**: Linux, macOS, Windows (cross-platform CLI binary)
**Project Type**: Single project (Go monorepo at `radius/`)
**Performance Goals**: Under 5 seconds for a typical 10-20 resource Bicep file (dominated by Bicep compilation)
**Constraints**: Fully offline (no Radius control plane API calls); `rad-bicep` compiler must be available locally or auto-downloadable
**Scale/Scope**: Single-file Bicep/ARM JSON templates; typical files contain 5-30 resources

### Key Existing Code

| Component | File | Signature/Role |
|-----------|------|----------------|
| CLI command | `pkg/cli/cmd/app/graph/graph.go` | `NewCommand()`, `Runner.Validate()`, `Runner.Run()` |
| Graph display | `pkg/cli/cmd/app/graph/display.go` | `display(resources, appName) string` |
| Graph builder | `pkg/corerp/frontend/controller/applications/graph_util.go` | `computeGraph(appResources, envResources) *ApplicationGraphResponse` |
| Source resolver | `pkg/corerp/frontend/controller/applications/graph_util.go` | `findSourceResource(source, allResources) (string, error)` |
| Bicep compiler | `pkg/cli/bicep/types.go` | `Interface.PrepareTemplate(filePath) (map[string]any, error)` |
| Bicep progress output | `cmd/rad/cmd/root.go` | Bicep `OutputWriter` wired to `RootCmd.ErrOrStderr()` to keep stdout clean |
| GenericResource | `pkg/cli/clients_new/generated/` | `GenericResource{ID, Name, Type, Properties}` |
| Graph types | `pkg/corerp/api/v20231001preview/` | `ApplicationGraphResponse`, `ApplicationGraphResource`, `ApplicationGraphConnection` |
| Output flags | `pkg/cli/cmd/commonflags/flags.go` | `AddOutputFlag(cmd)` — registers `--output` / `-o` |
| Output format | `pkg/cli/clivalidation.go` | `RequireOutput(cmd) (string, error)` — reads and validates |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. API-First Design | **N/A** | No new API endpoints; this is a client-side-only feature reusing existing graph types |
| II. Idiomatic Code Standards | **PASS** | Go implementation following Effective Go; godoc on exports; `gofmt` formatting |
| III. Multi-Cloud Neutrality | **PASS** | Fully cloud-agnostic; operates on Bicep/ARM JSON locally without cloud dependencies |
| IV. Testing Pyramid Discipline | **PASS** | Unit tests for template extraction, ARM expression resolution, synthetic ID generation; integration tests with real Bicep files |
| V. Collaboration-Centric Design | **PASS** | Serves developers (design-time graph preview) and platform engineers (CI validation) |
| VI. Open Source and Community-First | **PASS** | Design spec in design-notes repo; implementation in radius repo |
| VII. Simplicity Over Cleverness | **PASS** | Reuses existing `computeGraph()` and `PrepareTemplate()`; no new abstraction layers; simple regex for ARM reference resolution |
| VIII. Separation of Concerns | **PASS** | Template-to-GenericResource conversion is a new isolated function; graph computation and display are reused unchanged |
| IX. Incremental Adoption | **PASS** | Additive `--file` flag; existing `rad app graph <name>` behavior unchanged |
| X-XI. TypeScript/Dashboard | **N/A** | No dashboard changes |
| XII-XIII. Resource Types/Recipes | **N/A** | No schema or recipe changes |
| XIV-XV. Documentation | **PASS** | CLI help text updated; docs repo update for `rad app graph` reference page |
| XVI. Repository-Specific Standards | **PASS** | Follows `radius` repo patterns for CLI commands, test structure |
| XVII. Polyglot Coherence | **PASS** | Single-repo change (radius); consistent error handling and output patterns |

**Gate result**: **PASS** — No violations. Proceed to Phase 0.

## Project Structure

### Documentation (this feature)

```text
specs/002-app-graph-file/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (radius repository)

```text
pkg/cli/cmd/app/graph/
├── graph.go             # Modified: add --file flag, branch in Validate/Run
├── graph_test.go        # Modified: add tests for --file mode
├── display.go           # Unchanged (reused for text rendering)
├── display_test.go      # Unchanged
├── display_dot.go       # NEW: Graphviz DOT format output
├── display_dot_test.go  # NEW: unit tests for DOT output
├── template.go          # NEW: ARM JSON → []GenericResource conversion
└── template_test.go     # NEW: unit tests for template extraction

pkg/corerp/frontend/controller/applications/
├── graph_util.go        # Modified: export computeGraph → ComputeGraph
└── graph_util_test.go   # Unchanged

cmd/rad/cmd/
└── root.go              # Modified: Bicep Output writer → ErrOrStderr()
```

**Structure Decision**: This feature adds four new files (`template.go`, `template_test.go`, `display_dot.go`, `display_dot_test.go`) in the existing `pkg/cli/cmd/app/graph/` package and modifies the existing `graph.go` and `graph_test.go`. No new packages or abstraction layers are introduced. The ARM template extraction logic is co-located with the graph command since it is specific to this feature's `--file` mode. The DOT display function follows the same pattern as the existing `display.go` for text output.

## Constitution Check — Post-Design Re-evaluation

*Re-evaluated after Phase 1 design completion (data-model.md, contracts/, quickstart.md).*

| Principle | Status | Post-Design Notes |
|-----------|--------|-------------------|
| I. API-First Design | **N/A** | No new API endpoints. Exporting `ComputeGraph()` is an internal Go API change, not a user-facing API. |
| II. Idiomatic Code Standards | **PASS** | Design follows Go idioms: exported function with godoc, `gofmt`, testify/require, gomock. |
| III. Multi-Cloud Neutrality | **PASS** | Fully offline, no cloud dependency. Operates on local Bicep/ARM JSON files. |
| IV. Testing Pyramid Discipline | **PASS** | Unit tests for `template.go` (expression resolution, ID synthesis, resource extraction). Integration tests for end-to-end `--file` mode. Test fixtures in `testdata/`. |
| V. Collaboration-Centric Design | **PASS** | Developers: design-time graph preview. Platform engineers: CI-based architecture validation. |
| VI. Open Source and Community-First | **PASS** | Design documented in `design-notes`. Implementation in public `radius` repo. |
| VII. Simplicity Over Cleverness | **PASS** | Regex for ARM expression resolution (not a full evaluator). Reuses `computeGraph()` and `display()` unchanged. No new abstraction layers. |
| VIII. Separation of Concerns | **PASS** | Template extraction isolated in `template.go`. Graph computation and display reused unchanged. `ComputeGraph()` export justified: pure computation function with no server-side dependencies (HTTP, DB, auth). `displayDot()` is used in both file-mode and live-mode `Run()` paths. |
| IX. Incremental Adoption | **PASS** | Additive `--file` flag. Existing `rad app graph <name>` behavior unchanged. `--workspace`/`--group` ignored silently in file mode. |
| X-XI. TypeScript/Dashboard | **N/A** | No dashboard changes. |
| XII-XIII. Resource Types/Recipes | **N/A** | No schema or recipe changes. |
| XIV-XV. Documentation | **PASS** | CLI help text updated via Cobra flag registration. Docs repo reference page update planned. |
| XVI. Repository-Specific Standards | **PASS** | Follows `radius` repo CLI command patterns (`NewCommand`, `Runner`, `Validate`, `Run`). Test fixtures in `testdata/`. |
| XVII. Polyglot Coherence | **PASS** | Single-repo change. Consistent error handling (`clierrors.Message`) and output patterns (`Output.WriteFormatted`). |

**Post-design gate result**: **PASS** — No new violations. Design is sound and aligned with all applicable constitution principles. The `ComputeGraph()` export is the only structural change to existing code, and it is minimal and well-justified.

## Complexity Tracking

> No violations to justify. All design decisions align with constitution principles.
