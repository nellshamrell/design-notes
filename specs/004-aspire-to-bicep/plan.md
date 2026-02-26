# Implementation Plan: Aspire-to-Bicep PoC CLI Command

**Branch**: `004-aspire-to-bicep` | **Date**: 2026-02-23 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/004-aspire-to-bicep/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Build a `rad bicep generate --from-aspire` CLI subcommand that reads `azd infra synth` output from the reference Aspire application (`./example-aspire-app` with webfrontend + apiservice + Redis + SQL Server) and produces a Radius-deployable `app.bicep` file plus a companion `mapping-report.md`. The input consists of per-service YAML templates (`.tmpl.yaml`) in the AppHost's `infra/` directory and solution-level Bicep files in the top-level `infra/` directory. The command is implemented in Go within the existing Radius CLI codebase using the Cobra/`framework.Factory` pattern. It uses a YAML parser for `.tmpl.yaml` files (with Go template expression stripping), a lightweight Bicep text parser for `main.bicep`, a two-phase discovery+extraction mapper, Go `text/template` for Bicep output generation, and golden-file tests for idempotency verification.

## Technical Context

**Language/Version**: Go 1.26.0 (`github.com/radius-project/radius`)
**Primary Dependencies**: Cobra v1.10.2, `framework.Factory` (Radius CLI framework), Go `text/template`, `gopkg.in/yaml.v3` (YAML parsing), `os`/`path/filepath` (file I/O), `regexp` (Go template expression stripping, Bicep parsing)
**Storage**: N/A — file-based I/O only (reads azd Bicep directory, writes `app.bicep` + `mapping-report.md`)
**Testing**: `go test` with table-driven unit tests, golden-file tests for idempotency, integration tests via `radcli` test helpers (`SharedCommandValidation`, `ValidateInput`); reference application fixtures from `./example-aspire-app`
**Target Platform**: Cross-platform CLI (Linux, macOS, Windows) — same as existing `rad` binary
**Project Type**: Single project — extends the existing `radius` monorepo CLI codebase
**Performance Goals**: N/A — batch conversion of a small number of files (< 10 YAML templates + Bicep files for PoC scope)
**Constraints**: Must integrate with existing `rad` CLI command structure; must produce valid Bicep output; no external runtime dependencies beyond the `rad` binary itself (no Bicep CLI, no .NET required at conversion time)
**Scale/Scope**: PoC scope — reference application topology (2 services + 1 Redis + 1 SQL Server); single Aspire project only

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Status | Notes |
|---|-----------|--------|-------|
| I | API-First Design | **PASS** | Not a new API — this is a CLI command. Internal interfaces (parser, mapper, generator, reporter) are well-defined in the data model. |
| II | Idiomatic Code Standards | **PASS** | Go code follows Effective Go; `gofmt` formatting; godoc comments on all exported items; table-driven tests. |
| III | Multi-Cloud Neutrality | **PASS** | Input is Azure-specific (azd Bicep), but output is cloud-neutral Radius resources (Portable Resources, Recipe-backed Redis). The conversion bridges Azure → Radius neutrality. |
| IV | Testing Pyramid Discipline | **PASS** | Unit tests (parser, mapper, generator, reporter), integration tests (CLI command validation via `radcli`), golden-file tests (idempotency). Functional tests documented as manual validation in quickstart.md. |
| V | Collaboration-Centric Design | **PASS** | The conversion bridges developer (Aspire) and platform engineer (Radius) workflows. Mapping report enables both audiences to understand the translation. |
| VI | Open Source and Community-First | **PASS** | Design spec in `design-notes` repo; implementation follows issue-first workflow. |
| VII | Simplicity Over Cleverness | **PASS** | Lightweight text parser (no full Bicep compiler); Go `text/template` for output; fixed PoC scope avoids over-engineering. |
| VIII | Separation of Concerns | **PASS** | Four-component pipeline: Parser → Mapper → Generator → Reporter. Each component has a single responsibility with clear interfaces defined in the data model. |
| IX | Incremental Adoption | **PASS** | New CLI subcommand — no breaking changes to existing commands or workflows. |
| XIV | Documentation Structure | **PASS** | Mapping report (FR-008/FR-009) provides structured documentation of every conversion decision. quickstart.md follows tutorial pattern. |
| XVI | Repository-Specific Standards | **PASS** | Follows `radius` repo conventions: `pkg/cli/cmd/bicep/` directory, `NewCommand`/`Runner` pattern, `make test`/`make lint`. |
| XVII | Polyglot Project Coherence | **PASS** | Single-repo change (radius CLI in Go). Output is Bicep, which is the standard Radius application authoring language. No cross-repo coordination needed. |

**Pre-Phase 0 Gate**: **PASSED** — No violations. All applicable principles satisfied.

## Project Structure

### Documentation (this feature)

```text
specs/004-aspire-to-bicep/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
│   └── cli-contract.md  # CLI command contract
├── checklists/          # Quality checklists
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (radius repository)

```text
pkg/cli/cmd/bicep/
├── publish/                          # Existing: rad bicep publish
├── publishextension/                 # Existing: rad bicep publish-extension
├── generatekubernetesmanifest/       # Existing: rad bicep generate-kubernetes-manifest
└── generate/                         # NEW: rad bicep generate --from-aspire
    ├── generate.go                   # Cobra command + Runner (NewCommand, Validate, Run)
    ├── generate_test.go              # Unit tests for command validation
    ├── parser.go                     # YAML template + Bicep file parser
    ├── parser_test.go                # Parser unit tests with testdata fixtures
    ├── mapper.go                     # Aspire → Radius model mapper
    ├── mapper_test.go                # Mapper unit tests
    ├── generator.go                  # Bicep output generator (text/template)
    ├── generator_test.go             # Generator unit tests + golden file tests
    ├── reporter.go                   # Mapping report generator (console + Markdown)
    ├── reporter_test.go              # Reporter unit tests
    ├── models.go                     # Data model types (Parsed Model + Radius Model + MappingReport)
    ├── templates/                    # Go text/template files
    │   ├── app.bicep.tmpl            # Template for app.bicep output
    │   └── mapping-report.md.tmpl    # Template for mapping-report.md output
    └── testdata/                     # Test fixtures
        ├── example-aspire-app/       # Reference application structure (from ./example-aspire-app)
        │   ├── infra/
        │   │   ├── main.bicep
        │   │   └── main.parameters.json
        │   └── AspireApp.AppHost/
        │       └── infra/
        │           ├── apiservice.tmpl.yaml
        │           ├── webfrontend.tmpl.yaml
        │           ├── cache.tmpl.yaml
        │           └── sqlserver.tmpl.yaml
        ├── golden/                   # Expected output files for golden tests
        │   ├── app.bicep
        │   └── mapping-report.md
        ├── empty/                    # Empty directory for error tests
        ├── missing-ports/            # Service with missing port info
        └── multi-project/            # Multiple AppHost infra dirs for error tests
```

**Structure Decision**: Extends the existing `radius` repository CLI structure. All new code lives under `pkg/cli/cmd/bicep/generate/` following the established one-package-per-command pattern (`publish/`, `publishextension/`, `generatekubernetesmanifest/`). Internal components (parser, mapper, generator, reporter) are package-private files within the same package to keep the PoC simple. Test fixtures replicate the `./example-aspire-app` directory structure (YAML templates + solution-level Bicep) using a `testdata/` directory.

## Complexity Tracking

> No violations found — this section is intentionally empty.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| (none) | — | — |
