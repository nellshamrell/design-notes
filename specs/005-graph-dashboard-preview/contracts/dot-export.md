# Contract: DOT Export

**Date**: 2026-03-05
**Feature**: [spec.md](../spec.md) | [data-model.md](../data-model.md)

## Overview

This contract defines the interface for exporting application graph data to Graphviz DOT format, consistent with the Radius CLI's `displayDot()` output.

## Export Function

### Function: `exportToDot`

```typescript
/**
 * Converts an ApplicationGraphResponse to Graphviz DOT format.
 * Output is consistent with the Radius CLI's `rad app graph --output dot`.
 *
 * @param response - The validated ApplicationGraphResponse
 * @param graphName - Name for the digraph (defaults to "radius")
 * @returns DOT format string
 */
function exportToDot(
  response: ApplicationGraphResponse,
  graphName?: string
): string;
```

## DOT Format Specification

### Structure

```dot
digraph "graphName" {
    rankdir=LR;
    node [style=filled, fontname="Helvetica"];

    "resourceName" [label="resourceName\n(resourceType)", shape=box, fillcolor=lightblue];
    ...

    "sourceName" -> "targetName";
    ...
}
```

### Node rules

| Condition | Shape | Fill color |
|-----------|-------|-----------|
| Radius resource (`type` starts with `Applications.` or `Microsoft.`) | `box` | `lightblue` |
| Non-Radius resource | `ellipse` | `lightyellow` |

- Label format: `"name\n(type)"`
- Node names are the resource `name` field, double-quote escaped

### Edge rules

- Only **outbound** connections produce edges
- Edges are **deduplicated** (same source→target pair appears only once)
- Target name is extracted from the connection `id` (last segment after final `/`)
- All edges are sorted alphabetically for deterministic output

### Resource ordering

- Resources are sorted by `type` (ascending), then `name` (ascending) for deterministic output

### Special characters

- Double quotes in resource names and types are escaped as `\"`

## Download/Clipboard Contract

### Function: `downloadDotFile`

```typescript
/**
 * Triggers a browser file download of the DOT content.
 *
 * @param dotContent - The DOT format string
 * @param filename - Download filename (defaults to "graph.dot")
 */
function downloadDotFile(dotContent: string, filename?: string): void;
```

### Function: `copyDotToClipboard`

```typescript
/**
 * Copies DOT content to the system clipboard.
 *
 * @param dotContent - The DOT format string
 * @returns Promise that resolves when copy is complete
 */
function copyDotToClipboard(dotContent: string): Promise<void>;
```
