# Internal Function Contract: `extractResourcesFromTemplate`

**Date**: 2026-03-03
**Feature**: [spec.md](../spec.md) | [data-model.md](../data-model.md)
**File**: `pkg/cli/cmd/app/graph/template.go`

## Primary Function

### `extractResourcesFromTemplate`

```go
func extractResourcesFromTemplate(template map[string]any) ([]generated.GenericResource, []string, error)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `template` | `map[string]any` | Parsed ARM JSON template (output of `PrepareTemplate()`) |

| Return | Type | Description |
|--------|------|-------------|
| `resources` | `[]generated.GenericResource` | Extracted resources with synthesized IDs and resolved properties |
| `warnings` | `[]string` | Warnings for unresolvable expressions, conditionals, modules |
| `error` | `error` | Non-nil only for fatal template structure errors (e.g., missing `resources` key) |

### Behavior Contract

1. Read `template["resources"]` as `map[string]any` (symbolic-name keyed)
2. For each resource entry:
   a. Extract `type` field, strip `@apiVersion` suffix
   b. Extract `properties.name` as resource name (fall back to symbolic name if missing)
   c. Synthesize resource ID: `/planes/radius/local/resourceGroups/default/providers/{Type}/{Name}`
   d. Detect Radius vs. non-Radius resource (via `import == "Radius"` or namespace prefix)
   e. For Radius resources: extract and resolve `properties.properties.application`, `connections`, `routes`
   f. For non-Radius resources: set minimal properties (`provisioningState`, `status`)
   g. Detect and warn on `condition` field, `Microsoft.Resources/deployments` type
3. Build `generated.GenericResource` for each entry
4. Return all resources, accumulated warnings, and nil error (or fatal error)

### Pre-conditions

- `template` is a valid ARM JSON template with `resources` key
- `template` was produced by `PrepareTemplate()` or parsed from valid JSON

### Post-conditions

- Every returned `GenericResource` has non-nil `ID`, `Name`, `Type`
- Every returned `GenericResource` has `Properties["provisioningState"] == "NotDeployed"`
- Every returned `GenericResource` has `Properties["status"]["outputResources"] == []`
- Resources are in deterministic order (sorted by type, then name)

## Helper Functions

### `resolveExpression`

```go
func resolveExpression(value string, resources map[string]resourceEntry) (string, bool)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `value` | `string` | Raw ARM expression or literal string |
| `resources` | `map[string]resourceEntry` | Symbolic name → resource entry lookup table |

| Return | Type | Description |
|--------|------|-------------|
| `resolved` | `string` | Resolved value (synthesized ID or literal pass-through) |
| `ok` | `bool` | `false` if expression is unresolvable (caller should warn) |

### Behavior

1. If `value` does not start with `[` → return `(value, true)` (literal pass-through)
2. If `value` matches `\[reference\('(\w+)'\)\.id\]` → extract symbolic name → look up in resources → return `(synthesizedID, true)` or `("", false)` if not found
3. Otherwise → return `("", false)` (unsupported expression)

### `synthesizeResourceID`

```go
func synthesizeResourceID(resourceType, resourceName string) string
```

Returns `/planes/radius/local/resourceGroups/default/providers/{resourceType}/{resourceName}`

### `stripAPIVersion`

```go
func stripAPIVersion(armType string) string
```

Strips `@apiVersion` suffix: `Applications.Core/containers@2023-10-01-preview` → `Applications.Core/containers`

### `isRadiusResource`

```go
func isRadiusResource(importField string, resourceType string) bool
```

Returns `true` if `importField == "Radius"` (case-insensitive) OR if `resourceType` starts with `Applications.` or `Radius.`

### `countApplicationResources`

```go
func countApplicationResources(resources map[string]resourceEntry) int
```

Counts resources whose type (after stripping API version) is `Applications.Core/applications` or `Radius.Core/applications`.

## Internal Type

### `resourceEntry`

```go
type resourceEntry struct {
    SymbolicName string
    Type         string         // with API version stripped
    Name         string         // from properties.name
    Import       string         // e.g., "Radius"
    Properties   map[string]any // inner properties (Radius properties)
    HasCondition bool
    SynthesizedID string
}
```

This is a transient struct used only within `template.go` for intermediate processing.
