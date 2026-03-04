# CLI Interface Contract: `rad app graph --file`

**Date**: 2026-03-03
**Feature**: [spec.md](../spec.md) | [plan.md](../plan.md)

## Command Signature

```
rad app graph [<app-name>] [--file <path>] [--output <format>] [--workspace <name>] [--group <name>]
```

### Flag: `--file` / `-f`

| Property | Value |
|----------|-------|
| Type | `string` |
| Required | No |
| Default | `""` (empty — live mode) |
| Description | Path to a `.bicep` or `.json` ARM template file for offline graph generation |

### Mutual Exclusivity

| `<app-name>` provided | `--file` provided | Behavior |
|------------------------|-------------------|----------|
| yes | no | Live mode (existing behavior) |
| no | yes | File mode (new behavior) |
| yes | yes | **Error**: `"--file and application name are mutually exclusive"` |
| no | no | Existing behavior (uses default app or prompts) |

### Output Formats

| `--output` value | File mode behavior |
|------------------|-------------------|
| `table` (default) | Text tree via `display()` |
| `json` | JSON via `Output.WriteFormatted()` using `ApplicationGraphResponse` schema |
| `dot` | Graphviz DOT language via `displayDot()` — valid input for `dot -Tpng` |

**Note**: The `dot` output format is available in both `--file` mode and live mode.

### Workspace/Group Flags in File Mode

When `--file` is provided, `--workspace` and `--group` are **ignored** (no API calls made). No error is raised if they are provided alongside `--file`.

## Exit Codes

| Scenario | Exit Code | Stderr |
|----------|-----------|--------|
| Success | 0 | Warnings (if any) |
| File not found | 1 | Error message |
| Bicep compilation error | 1 | Compiler error output |
| Multiple applications in file | 1 | Error with guidance |
| `--file` + `<app-name>` conflict | 1 | Mutual exclusivity error |
| No Radius resources in file | 0 | Empty graph message |

## Warnings (stderr)

| Condition | Warning message pattern |
|-----------|------------------------|
| Unresolvable ARM expression | `Warning: cannot resolve connection source "{expression}" for resource "{name}": unsupported expression` |
| Conditional resource | `Warning: resource "{name}" has a condition and may not be deployed` |
| Module reference | `Warning: module reference "{name}" detected; nested resources are not included in the graph` |
| Duplicate resource | `Warning: duplicate resource detected: {type}/{name}` |
