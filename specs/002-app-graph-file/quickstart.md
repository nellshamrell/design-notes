# Quickstart: `rad app graph --file` Static Graph from Bicep

**Feature**: 002-app-graph-file
**Date**: 2026-03-03

## Overview

This document provides a quick reference for implementing the `--file` flag on `rad app graph` that generates an application graph directly from a Bicep file, without deploying to a Radius environment.

## High-Level Changes

### 1. New File: `pkg/cli/cmd/app/graph/template.go`

Contains all template extraction logic:

- `extractResourcesFromTemplate(template map[string]any) ([]generated.GenericResource, []string, error)` — main extraction function
- `resolveExpression(value string, resources map[string]resourceEntry) (string, bool)` — ARM expression resolver
- `synthesizeResourceID(resourceType, resourceName string) string` — ID generator
- `stripAPIVersion(armType string) string` — strips `@apiVersion` suffix
- `isRadiusResource(importField, resourceType string) bool` — Radius resource detector
- `countApplicationResources(resources map[string]resourceEntry) int` — application counter
- `resourceEntry` struct — transient intermediate type

### 2. Modify: `pkg/cli/cmd/app/graph/graph.go`

- Add `FilePath string` and `BicepClient bicep.Interface` fields to `Runner` struct
- Register `--file` / `-f` flag in `NewCommand()`
- Branch in `Validate()`: mutual exclusivity check, file existence validation
- Branch in `Run()`: compile → extract → computeGraph → display/JSON

### 3. Export: `pkg/corerp/frontend/controller/applications/graph_util.go`

- Rename `computeGraph()` → `ComputeGraph()` to export it for CLI reuse

### 4. New File: `pkg/cli/cmd/app/graph/template_test.go`

Unit tests for all `template.go` functions.

### 5. Modify: `pkg/cli/cmd/app/graph/graph_test.go`

Integration tests for `--file` mode end-to-end flows.

### 6. New File: `pkg/cli/cmd/app/graph/display_dot.go`

Graphviz DOT format output function:

- `displayDot(resources []*ApplicationGraphResource, appName string) string` — produces a valid Graphviz DOT digraph string

### 7. New File: `pkg/cli/cmd/app/graph/display_dot_test.go`

Unit tests for `displayDot()` function.

## Implementation Steps

### Step 1: Export `computeGraph`

```go
// In pkg/corerp/frontend/controller/applications/graph_util.go
// Rename computeGraph → ComputeGraph
func ComputeGraph(applicationResources []generated.GenericResource, environmentResources []generated.GenericResource) *corerpv20231001preview.ApplicationGraphResponse {
```

Update all internal callers in the same package.

### Step 2: Create `template.go`

```go
package graph

import (
    "fmt"
    "regexp"
    "sort"
    "strings"

    "github.com/radius-project/radius/pkg/cli/clients_new/generated"
    "github.com/radius-project/radius/pkg/to"
)

var referencePattern = regexp.MustCompile(`\[reference\('(\w+)'\)\.id\]`)

type resourceEntry struct {
    SymbolicName  string
    Type          string
    Name          string
    Import        string
    Properties    map[string]any
    HasCondition  bool
    SynthesizedID string
}

func extractResourcesFromTemplate(template map[string]any) ([]generated.GenericResource, []string, error) {
    // See contracts/template-extraction.md for full behavior contract
}
```

### Step 3: Add `--file` flag to Runner

```go
// In NewCommand():
cmd.Flags().StringP("file", "f", "", "Path to a .bicep or .json file for offline graph generation")

// In Runner struct:
FilePath    string
BicepClient bicep.Interface
```

### Step 4: Branch in `Validate()`

```go
r.FilePath, _ = cmd.Flags().GetString("file")

if r.FilePath != "" && len(args) > 0 {
    return clierrors.Message("--file and application name are mutually exclusive")
}

if r.FilePath != "" {
    // Validate file exists
    if _, err := os.Stat(r.FilePath); err != nil {
        return clierrors.Message("file not found: %s", r.FilePath)
    }
    r.Format, err = cli.RequireOutput(cmd)
    if err != nil {
        return err
    }
    return nil
}
// ... existing live-mode validation
```

### Step 5: Branch in `Run()`

```go
if r.FilePath != "" {
    template, err := r.BicepClient.PrepareTemplate(r.FilePath)
    if err != nil {
        return err
    }
    
    resources, warnings, err := extractResourcesFromTemplate(template)
    if err != nil {
        return err
    }
    
    for _, w := range warnings {
        fmt.Fprintln(os.Stderr, "Warning:", w)
    }
    
    // Application scoping
    appName, appResources, err := scopeToApplication(resources)
    if err != nil {
        return err
    }
    
    response := applications.ComputeGraph(appResources, nil)
    
    // Output
    switch r.Format {
    case "json":
        return r.Output.WriteFormatted(r.Format, response, ...)
    case "dot":
        r.Output.LogInfo(displayDot(response.Resources, appName))
        return nil
    default:
        r.Output.LogInfo(display(response.Resources, appName))
        return nil
    }
}
```

## Key Commands Reference

| Action | Command |
|--------|---------|
| Generate graph from Bicep | `rad app graph --file app.bicep` |
| Generate JSON graph | `rad app graph --file app.bicep --output json` |
| Generate DOT graph | `rad app graph --file app.bicep --output dot` |
| Render DOT to PNG | `rad app graph --file app.bicep --output dot \| dot -Tpng -o graph.png` |
| Generate from ARM JSON | `rad app graph --file template.json` |
| Existing live graph | `rad app graph myapp` |
| Live graph as DOT | `rad app graph myapp --output dot` |

## Test Fixtures

Create minimal test fixtures in `pkg/cli/cmd/app/graph/testdata/`:

| File | Purpose |
|------|---------|
| `simple-app.json` | ARM JSON with 1 app, 2 containers, 1 connection |
| `no-app.json` | ARM JSON with resources but no application resource |
| `multi-app.json` | ARM JSON with 2 application resources (error case) |
| `non-radius.json` | ARM JSON with mixed Radius and Azure resources |
| `unresolvable.json` | ARM JSON with parameterized connection sources |
| `empty.json` | ARM JSON with empty resources map |
| `with-modules.json` | ARM JSON with `Microsoft.Resources/deployments` entries |

## Success Verification

After implementation, verify:

1. `rad app graph --file app.bicep` produces same-format output as live `rad app graph`
2. `rad app graph --file app.bicep --output json` produces valid JSON matching the live schema
3. `rad app graph --file app.bicep --output dot` produces valid Graphviz DOT that renders with `dot -Tpng`
4. `rad app graph myapp --file app.bicep` errors with mutual exclusivity message
5. `rad app graph --file nonexistent.bicep` errors with file-not-found message
6. Bicep syntax errors are surfaced clearly
7. Unresolvable ARM expressions produce warnings, not errors
8. Non-Radius resources appear as graph nodes
9. Non-Radius resources use different node shapes in DOT output
10. Output is deterministic (same input → same output)
11. No network calls are made in `--file` mode
