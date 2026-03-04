# Quickstart: `rad app graph --file` Static Graph from Bicep

**Feature**: 002-app-graph-file
**Date**: 2026-03-03

## Overview

This document provides a quick reference for implementing the `--file` flag on `rad app graph` that generates an application graph directly from a Bicep file, without deploying to a Radius environment.

## Implementation Progress

| Item | Status | Notes |
|------|--------|-------|
| Export `ComputeGraph` | ✅ Done | `pkg/corerp/frontend/controller/applications/graph_util.go` |
| Test fixtures (`testdata/`) | ✅ Done | 7 fixture files created |
| `template.go` | ✅ Done | All helper functions and `extractResourcesFromTemplate` |
| `template_test.go` | ✅ Done | Unit tests for all template functions |
| `--file` flag + `Runner` fields | ✅ Done | `graph.go` |
| `Validate()` file mode branch | ✅ Done | Mutual exclusivity, file existence check |
| `runFileMode()` text + JSON output | ✅ Done | `graph.go` |
| `runFileMode()` DOT output | ✅ Done | `case output.FormatDot` in `runFileMode` switch |
| `runLiveMode()` DOT output | ✅ Done | `case output.FormatDot` in `runLiveMode` switch |
| `display_dot.go` | ✅ Done | `displayDot`, `escapeDot`, `resourceNameFromID` |
| `display_dot_test.go` | ✅ Done | 7 unit tests covering all display_dot functions |
| Integration tests (file mode DOT) | ✅ Done | `Test_Run_FileMode_Dot` in `graph_test.go` |
| Integration tests (live mode DOT) | ✅ Done | `Test_Run_Dot` in `graph_test.go` |
| Errors routed to stderr | ✅ Done | `cmd/rad/cmd/root.go` `Execute()` + `handlePanic()` |
| Bicep progress output to stderr | ✅ Done | `cmd/rad/cmd/root.go` Bicep `OutputWriter` uses `ErrOrStderr()` |

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
- Branch in `Run()`: file mode compiles → extracts → computeGraph → text/JSON/DOT output; live mode adds a `case output.FormatDot` to the existing format switch calling `displayDot()`

### 3. Export: `pkg/corerp/frontend/controller/applications/graph_util.go`

- Rename `computeGraph()` → `ComputeGraph()` to export it for CLI reuse

### 4. New File: `pkg/cli/cmd/app/graph/template_test.go`

Unit tests for all `template.go` functions.

### 5. Modify: `pkg/cli/cmd/app/graph/graph_test.go`

Integration tests for `--file` mode end-to-end flows.

### 6. New File: `pkg/cli/cmd/app/graph/display_dot.go`

Graphviz DOT format output function:

- `displayDot(resources []*ApplicationGraphResource, appName string) string` — produces a valid Graphviz DOT digraph string
- `escapeDot(s string) string` — escapes double quotes in DOT string values
- `resourceNameFromID(id string) string` — extracts the resource name from a Radius resource ID for use as edge target
- Radius resources rendered as `shape=box, fillcolor=lightblue`; non-Radius resources as `shape=ellipse, fillcolor=lightyellow`
- Edges are outbound-only, deduplicated, and sorted for deterministic output
- Reuses `isRadiusResource()` from `template.go` for Radius vs non-Radius node styling

### 7. New File: `pkg/cli/cmd/app/graph/display_dot_test.go`

Unit tests for `displayDot()`, `escapeDot()`, and `resourceNameFromID()` functions.

### 8. Modify: `cmd/rad/cmd/root.go`

- Change Bicep client `Output` writer from `RootCmd.OutOrStdout()` to `RootCmd.ErrOrStderr()` so that Bicep progress messages (`"Building ..."`, `"Downloading Bicep ..."`) go to stderr instead of stdout. This prevents progress text from corrupting piped machine-readable output (DOT, JSON).

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

### Step 6: Create `display_dot.go`

Graphviz DOT format output function for visual graph rendering. Available in both `--file` mode and live mode.

```go
package graph

import (
    "fmt"
    "sort"
    "strings"

    "github.com/radius-project/radius/pkg/corerp/api/v20231001preview"
)

// displayDot produces a Graphviz DOT language digraph string from the application graph.
func displayDot(resources []*v20231001preview.ApplicationGraphResource, appName string) string {
    output := &strings.Builder{}
    graphName := appName
    if graphName == "" {
        graphName = "radius"
    }
    output.WriteString(fmt.Sprintf("digraph %q {\n", graphName))
    output.WriteString("    rankdir=LR;\n")
    output.WriteString("    node [style=filled, fontname=\"Helvetica\"];\n\n")

    // Sort resources for deterministic output
    sorted := make([]*v20231001preview.ApplicationGraphResource, len(resources))
    copy(sorted, resources)
    sort.Slice(sorted, func(i, j int) bool {
        if *sorted[i].Type != *sorted[j].Type {
            return *sorted[i].Type < *sorted[j].Type
        }
        return *sorted[i].Name < *sorted[j].Name
    })

    // Emit nodes
    for _, r := range sorted {
        name := escapeDot(*r.Name)
        typeName := escapeDot(*r.Type)
        if isRadiusResource("", *r.Type) {
            output.WriteString(fmt.Sprintf("    %q [label=\"%s\\n(%s)\", shape=box, fillcolor=lightblue];\n",
                name, name, typeName))
        } else {
            output.WriteString(fmt.Sprintf("    %q [label=\"%s\\n(%s)\", shape=ellipse, fillcolor=lightyellow];\n",
                name, name, typeName))
        }
    }
    if len(sorted) > 0 {
        output.WriteString("\n")
    }

    // Emit edges (outbound connections only, deduplicated)
    seen := map[string]bool{}
    var edges []string
    for _, r := range sorted {
        for _, conn := range r.Connections {
            if *conn.Direction != v20231001preview.DirectionOutbound {
                continue
            }
            // Find target resource name
            targetName := resourceNameFromID(*conn.ID)
            edgeKey := *r.Name + "->" + targetName
            if seen[edgeKey] {
                continue
            }
            seen[edgeKey] = true
            edges = append(edges, fmt.Sprintf("    %q -> %q;\n",
                escapeDot(*r.Name), escapeDot(targetName)))
        }
    }
    sort.Strings(edges)
    for _, e := range edges {
        output.WriteString(e)
    }

    output.WriteString("}\n")
    return output.String()
}
```

**Design decisions**:
- `rankdir=LR`: left-to-right layout for natural data-flow reading order
- Radius resources: `shape=box`, `fillcolor=lightblue`
- Non-Radius resources: `shape=ellipse`, `fillcolor=lightyellow`
- Edges deduplicated by `source->target` key
- All output sorted for determinism

## Known Issues (Resolved)

### ~~Error output goes to stdout (affects pipe scenarios)~~ — FIXED

**Status**: Resolved in two changes:

1. **T028a** (errors): Changed all `fmt.Println`/`fmt.Printf` in `Execute()` and `handlePanic()` in `cmd/rad/cmd/root.go` to write to `os.Stderr`.
2. **Bicep progress output**: Changed the Bicep client's `Output` writer from `RootCmd.OutOrStdout()` to `RootCmd.ErrOrStderr()` in `cmd/rad/cmd/root.go` `initSubCommands()`. This prevents `PrepareTemplate()` progress messages (`"Building app.bicep..."`, `"Downloading Bicep for channel ..."`) from being written to stdout, which would corrupt piped machine-readable output.

**Root cause**: The Bicep `Impl` struct's `Output` field was an `OutputWriter` backed by stdout. When `PrepareTemplate()` called `Output.BeginStep("Building %s...", filePath)`, the progress text was written to stdout — mixed into the DOT/JSON stream. When piping to `dot -Tpng`, Graphviz would fail with a syntax error because the first line was `Building app.bicep...` instead of `digraph`.

**Fix** (in `cmd/rad/cmd/root.go`):
```go
// Before:
Bicep: &bicep.Impl{
    FileSystem: filesystem.OSFileSystem{},
    Output:     &output.OutputWriter{Writer: RootCmd.OutOrStdout()},
},

// After:
Bicep: &bicep.Impl{
    FileSystem: filesystem.OSFileSystem{},
    Output:     &output.OutputWriter{Writer: RootCmd.ErrOrStderr()},
},
```

**Debugging checklist if the pipe fails**:
1. Does `app.bicep` exist in the current directory?
2. Is the bicep CLI installed? (`az bicep version` or `bicep --version`)
3. Does the file compile cleanly? (`az bicep build --file app.bicep`)
4. Is the `rad` binary up to date? (`make build` in the radius repo)

## Key Commands Reference

| Action | Command |
|--------|---------|
| Generate graph from Bicep | `rad app graph --file app.bicep` |
| Generate JSON graph | `rad app graph --file app.bicep --output json` |
| Generate DOT graph | `rad app graph --file app.bicep --output dot` |
| Render DOT to PNG | `rad app graph --file app.bicep --output dot \| dot -Tpng -o graph.png` |
| Render DOT to SVG | `rad app graph --file app.bicep --output dot \| dot -Tsvg -o graph.svg` |
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
