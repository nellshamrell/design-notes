# Implementation Plan: Aspire Manifest to Bicep Conversion

**Branch**: `001-aspire-to-bicep` | **Date**: 2026-02-19 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-aspire-to-bicep/spec.md`

## Summary

New `rad aspire convert` CLI command that reads an Aspire manifest JSON file and produces a Radius-compatible `app.bicep` file. The command maps Aspire container resources, backing services (Redis, PostgreSQL, MySQL), and external bindings to their Radius Bicep equivalents. Resources that the Aspire manifest publisher could not generate (entries with an `error` field instead of a `type` field) are gracefully skipped with warnings. Build-only container resources (`build.buildOnly: true`) are excluded from conversion entirely as they are build-time artifacts, not runtime containers. Implemented in Go following the existing Radius CLI `framework.Runner` pattern with Cobra commands, fitting into the `radius` repository's `pkg/cli/cmd/` structure.

## Technical Context

**Language/Version**: Go 1.26.0 (per `go.mod`)
**Primary Dependencies**: `github.com/spf13/cobra` (CLI framework), `github.com/radius-project/radius/pkg/cli/framework` (Runner pattern), `github.com/radius-project/radius/pkg/cli/filesystem` (file I/O), `github.com/radius-project/radius/pkg/cli/output` (console output)
**Storage**: N/A — file-to-file conversion (reads JSON, writes `.bicep`)
**Testing**: `go test` via `make test`; unit tests with table-driven patterns; `framework.MockFactory` for dependency injection; golden file comparisons for Bicep output
**Target Platform**: Cross-platform CLI (Linux, macOS, Windows) — same as existing `rad` CLI
**Project Type**: Single project — new package within existing `radius` monorepo
**Performance Goals**: Sub-second conversion for manifests with up to 50 resources
**Constraints**: No network access required; pure file transformation; output must compile with Radius Bicep toolchain; must handle errored manifest entries (resources with `error` field, no `type`) gracefully; must exclude build-only containers (`build.buildOnly: true`) from conversion
**Scale/Scope**: Conversion of manifests with 1-50 Aspire resources; ~5-7 new Go source files, ~3-4 test files

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| **I. API-First Design** | ✅ PASS | No new API endpoints — this is a CLI-only file transformation tool. Consumes an existing format (Aspire manifest) and produces an existing format (Radius Bicep). |
| **II. Idiomatic Code Standards** | ✅ PASS | Will follow Go conventions: `gofmt`, godoc comments on all exported items, error handling per *Effective Go*. |
| **III. Multi-Cloud Neutrality** | ✅ PASS | The conversion tool itself is cloud-neutral. Cloud-specific Aspire resources are explicitly out of scope (v1) and emit warnings. |
| **IV. Testing Pyramid Discipline** | ✅ PASS | Plan includes unit tests for each converter component, table-driven tests for mapping logic, golden file tests for end-to-end Bicep output validation. |
| **V. Collaboration-Centric Design** | ✅ PASS | Developer experience: simplifies Aspire→Radius migration. Platform engineer experience: generated Bicep uses standard Radius patterns (environments, connections, recipes). |
| **VI. Open Source and Community-First** | ✅ PASS | Spec authored in design-notes repo; feature discussed before implementation. |
| **VII. Simplicity Over Cleverness** | ✅ PASS | Direct mapping table approach — no plugin system, no reflection, no AST manipulation. Mapping table is a simple Go map. |
| **VIII. Separation of Concerns** | ✅ PASS | Clear separation: manifest parsing → resource mapping → Bicep generation. Each is a distinct package/file. |
| **IX. Incremental Adoption** | ✅ PASS | New command — additive, no breaking changes. Unsupported resources and errored manifest entries warn rather than fail. |
| **XVI. Repository-Specific Standards** | ✅ PASS | Follows existing `radius` repo CLI patterns: framework.Runner, Cobra commands, filesystem abstraction. |
| **XVII. Polyglot Project Coherence** | ✅ PASS | Single-repo change (radius). No cross-repo impact on dashboard, docs, or resource-types-contrib (docs update would be a follow-up). |

| **XIV. Documentation Structure** | ✅ PASS | CLI command help text follows Cobra conventions. A follow-up docs PR in the `docs` repo will be needed to document the `rad aspire convert` command (not in scope of this plan but noted). |
| **XV. Documentation Contribution Standards** | ✅ PASS | CLI docs auto-generation from Cobra definitions already exists in the Radius repo. The new `aspire` command group will automatically be picked up by the existing doc generation pipeline via Cobra's `GenMarkdownTree`. |

**Gate result (post-design re-evaluation)**: ✅ ALL PASS — no violations after Phase 1 design. No new complexity concerns from data model, CLI contract, or project structure decisions.

## Project Structure

### Documentation (this feature)

```text
specs/001-aspire-to-bicep/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output
│   └── cli-interface.md # Command contract (flags, args, output)
└── tasks.md             # Phase 2 output (NOT created by /speckit.plan)
```

### Source Code (radius repository)

```text
# New files for this feature (in the radius repo)
pkg/cli/cmd/aspire/
├── aspire.go                      # Parent cobra command: `rad aspire`
├── convert/
│   ├── convert.go                 # NewCommand + Runner (Validate/Run)
│   ├── convert_test.go            # Unit tests for command wiring & validation
│   ├── manifest.go                # Aspire manifest JSON parser (types + Parse)
│   ├── manifest_test.go           # Parser unit tests
│   ├── mapper.go                  # Resource mapping engine (Aspire → Bicep IR)
│   ├── mapper_test.go             # Mapping logic unit tests (table-driven)
│   ├── emitter.go                 # Bicep text emitter (IR → .bicep string)
│   ├── emitter_test.go            # Emitter unit tests + golden file comparisons
│   └── testdata/
│       ├── aspire-manifest.json   # Sample input (copy of repo root file)
│       ├── expected-basic.bicep   # Golden file: basic conversion
│       └── expected-full.bicep    # Golden file: full sample manifest

# Modified files
cmd/rad/cmd/root.go                # Wire aspireCmd into RootCmd + initSubCommands
```

**Structure Decision**: Follows the existing `pkg/cli/cmd/{group}/{subcommand}/` convention used by `deploy`, `bicep/generatekubernetesmanifest`, `recipe`, etc. The `aspire` parent command mirrors `bicep.go` (thin Cobra group). The `convert` subpackage follows the `Runner` pattern (NewCommand, Validate, Run) matching every other CLI command in the repo.

## Complexity Tracking

> No constitution violations — this section is intentionally empty.
