# design-notes Development Guidelines

Auto-generated from all feature plans. Last updated: 2025-12-15

## Active Technologies
- Go 1.26.0 (per `go.mod`) + `github.com/spf13/cobra` (CLI framework), `github.com/radius-project/radius/pkg/cli/framework` (Runner pattern), `github.com/radius-project/radius/pkg/cli/filesystem` (file I/O), `github.com/radius-project/radius/pkg/cli/output` (console output) (001-aspire-to-bicep)
- N/A — file-to-file conversion (reads JSON, writes `.bicep`) (001-aspire-to-bicep)

- YAML (GitHub Actions), Bash + Radius CLI (installed via official installer), `rad version`, `rad upgrade kubernetes`, `rad install kubernetes` (001-lrt-current-release)

## Project Structure

```text
src/
tests/
```

## Commands

# Add commands for YAML (GitHub Actions), Bash

## Code Style

YAML (GitHub Actions), Bash: Follow standard conventions

## Recent Changes
- 001-aspire-to-bicep: Added Go 1.26.0 (per `go.mod`) + `github.com/spf13/cobra` (CLI framework), `github.com/radius-project/radius/pkg/cli/framework` (Runner pattern), `github.com/radius-project/radius/pkg/cli/filesystem` (file I/O), `github.com/radius-project/radius/pkg/cli/output` (console output)

- 001-lrt-current-release: Added YAML (GitHub Actions), Bash + Radius CLI (installed via official installer), `rad version`, `rad upgrade kubernetes`, `rad install kubernetes`

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
