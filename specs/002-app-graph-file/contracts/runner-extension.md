# Runner Extension Contract: File Mode

**Date**: 2026-03-03
**Feature**: [spec.md](../spec.md) | [plan.md](../plan.md)
**File**: `pkg/cli/cmd/app/graph/graph.go`

## Modified Runner Struct

```go
type Runner struct {
    ConfigHolder      *framework.ConfigHolder
    ConnectionFactory connections.Factory
    Output            output.Interface
    BicepClient       bicep.Interface  // NEW: injected for testability

    ApplicationName string
    Format          string
    Workspace       *workspaces.Workspace
    FilePath        string  // NEW: populated from --file flag
}
```

## Modified `NewCommand()`

```go
func NewCommand(factory framework.Factory) (*cobra.Command, framework.Runner)
```

**Changes**:
- Register `--file` / `-f` flag: `cmd.Flags().StringP("file", "f", "", "Path to a .bicep or .json file for offline graph generation")`
- Keep `cobra.MaximumNArgs(1)` (positional app name still optional)

## Modified `Validate()`

```go
func (r *Runner) Validate(cmd *cobra.Command, args []string) error
```

**New branching logic**:

```
1. Read --file flag value → r.FilePath
2. If r.FilePath != "" AND len(args) > 0:
   → return error: "--file and application name are mutually exclusive"
3. If r.FilePath != "":
   → Validate file exists and is readable
   → Read --output flag → r.Format
   → Skip workspace/scope/application validation
   → return nil
4. Else:
   → Execute existing live-mode validation unchanged
```

## Modified `Run()`

```go
func (r *Runner) Run(ctx context.Context) error
```

**New branching logic**:

```
1. If r.FilePath != "":
   → template, err := r.BicepClient.PrepareTemplate(r.FilePath)
   → resources, warnings, err := extractResourcesFromTemplate(template)
   → Emit warnings to stderr
   → Count application resources:
      - 0 apps: appName = "" (implicit), pass all resources
      - 1 app: appName = app name, filter to referencing resources
      - 2+ apps: return error
   → response := computeGraph(appResources, nil)  // nil env resources
   → Switch on r.Format:
      - json: r.Output.WriteFormatted(r.Format, response, ...)
      - dot: log displayDot(response.Resources, appName)
      - default: log display(response.Resources, appName)
   → return nil
2. Else:
   → Execute existing live-mode Run() unchanged
```

## Dependency Injection

The `BicepClient` field enables mock-based testing without invoking the real Bicep compiler.

- **Production**: `NewRunner()` initializes `BicepClient` via `factory.GetBicep()`. The Bicep `Impl` is configured in `cmd/rad/cmd/root.go` `initSubCommands()` with its `Output` writer set to `RootCmd.ErrOrStderr()` so that progress messages (`"Building ..."`, `"Downloading Bicep ..."`) go to stderr and don't pollute piped stdout output (DOT, JSON).
- **Tests**: Inject `mock_bicep.MockInterface` returning pre-built ARM JSON templates

## `computeGraph` Visibility

`computeGraph()` is currently unexported in `pkg/corerp/frontend/controller/applications/graph_util.go`. Two options:

| Option | Approach | Trade-off |
|--------|----------|-----------|
| **A: Export** | Rename to `ComputeGraph()` | Clean but changes server-side package surface |
| **B: Copy** | Copy `computeGraph()` into `pkg/cli/cmd/app/graph/` | Code duplication but no server-side changes |
| **C: Shared package** | Move to `pkg/graph/` shared package | Best separation but introduces new package |

**Recommended**: Option A (export). The function is a pure computation with no server-side dependencies. Exporting it makes it reusable without duplication.
