# design-notes Development Guidelines

Auto-generated from all feature plans. Last updated: 2025-12-15

## Active Technologies
- Go 1.26.0 (`github.com/radius-project/radius`) + Cobra v1.10.2, `framework.Factory` (Radius CLI framework), Go `text/template`, `os`/`path/filepath` (file I/O), `regexp` (Bicep parsing) (004-aspire-to-bicep)
- N/A — file-based I/O only (reads azd Bicep directory, writes `app.bicep` + `mapping-report.md`) (004-aspire-to-bicep)

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
- 004-aspire-to-bicep: Added Go 1.26.0 (`github.com/radius-project/radius`) + Cobra v1.10.2, `framework.Factory` (Radius CLI framework), Go `text/template`, `os`/`path/filepath` (file I/O), `regexp` (Bicep parsing)

- 001-lrt-current-release: Added YAML (GitHub Actions), Bash + Radius CLI (installed via official installer), `rad version`, `rad upgrade kubernetes`, `rad install kubernetes`

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
