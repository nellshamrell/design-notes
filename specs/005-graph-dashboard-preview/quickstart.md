# Quickstart: View Static Bicep Application Graphs in the Radius Dashboard

**Feature**: 005-graph-dashboard-preview
**Date**: 2026-03-05

## Overview

This document provides a quick reference for implementing the preview graph feature in the Radius Dashboard. The feature adds a dedicated `/preview` page where users can import `ApplicationGraphResponse` JSON (from the Radius CLI) and visualize it using the existing graph component, with distinct "Preview" styling.

## High-Level Changes

### 1. New: `packages/rad-components/src/lib/graphImport.ts`

Validation, transformation, and parsing utilities for `ApplicationGraphResponse` JSON:
- `validateApplicationGraphResponse(data: unknown)` — schema validation
- `transformToAppGraph(response, name?)` — converts API format to dashboard `AppGraph`
- `parseGraphJson(text: string)` — combines JSON.parse + validation

### 2. New: `packages/rad-components/src/lib/graphImport.test.ts`

Unit tests for all import/validation/transformation functions.

### 3. New: `packages/rad-components/src/lib/dotExport.ts`

DOT format export matching the Radius CLI's output:
- `exportToDot(response, graphName?)` — generates DOT string
- `downloadDotFile(dotContent, filename?)` — triggers browser download
- `copyDotToClipboard(dotContent)` — copies to clipboard

### 4. New: `packages/rad-components/src/lib/dotExport.test.ts`

Unit tests for DOT export functions.

### 5. New: `packages/rad-components/src/lib/shareableUrl.ts`

URL encoding/decoding for shareable graph links:
- `encodeGraphUrl(response, baseUrl)` — compress + encode to URL
- `decodeGraphUrl(hash)` — decode + validate from URL hash
- `copyShareUrl(response)` — encode + clipboard + update hash

### 6. New: `packages/rad-components/src/lib/shareableUrl.test.ts`

Unit tests for URL encoding/decoding.

### 7. New: `packages/rad-components/src/components/graphimport/GraphImportPanel.tsx`

React component for the import UI:
- JSON text area with paste support
- File upload dropzone (`.json` files)
- Error display for validation failures
- Loading state management

### 8. New: `packages/rad-components/src/components/graphimport/GraphImportPanel.test.tsx`

Unit tests for GraphImportPanel component.

### 9. New: `packages/rad-components/src/components/graphimport/__docs__/GraphImportPanel.stories.tsx`

Storybook stories for GraphImportPanel (empty, with error, with data).

### 10. Modify: `packages/rad-components/src/components/appgraph/AppGraph.tsx`

Add support for preview styling:
- Accept optional `isPreview` prop
- When `isPreview=true`: dashed node borders, muted colors, dashed edges (not animated)
- Pass `isPreview` to `ResourceNode`

### 11. Modify: `packages/rad-components/src/components/resourcenode/ResourceNode.tsx`

Add preview-specific styling:
- Accept `isPreview` from node data
- When preview: dashed border, muted background, "Not Deployed" badge

### 12. New: `packages/rad-components/src/components/previewbanner/PreviewBanner.tsx`

Banner component showing "Preview" indicator:
- Prominent banner above the graph
- Distinct styling (color, icon)
- Brief explanation text

### 13. New: `packages/rad-components/src/components/graphactions/GraphActions.tsx`

Action bar with export/share buttons:
- "Export as DOT" button (download)
- "Copy DOT to clipboard" button
- "Share" / "Copy Link" button
- Appears only when a graph is loaded

### 14. New: `plugins/plugin-radius/src/components/preview/PreviewPage.tsx`

The main preview page component:
- Import panel (left/top)
- Graph display (when data loaded) with preview styling
- Preview banner
- Action bar (export, share)
- Empty state / error state handling
- Large graph warning
- Reads URL hash on mount for shared links

### 15. New: `plugins/plugin-radius/src/components/preview/PreviewPage.test.tsx`

Unit tests for PreviewPage.

### 16. Modify: `plugins/plugin-radius/src/routes.ts`

Add new route ref:
- `previewPageRouteRef` — `'radius-preview-page'`

### 17. Modify: `plugins/plugin-radius/src/plugin.ts`

Add new routable extension:
- `PreviewPage` — lazy-loaded, bound to `previewPageRouteRef`

### 18. Modify: `plugins/plugin-radius/src/index.ts`

Export the new `PreviewPage` component and `previewPageRouteRef`.

### 19. Modify: `packages/app/src/App.tsx`

Add route mapping:
- `<Route path="/preview" element={<PreviewPage />} />`

### 20. Modify: `packages/app/src/components/Root/Root.tsx`

Add sidebar navigation entry:
- New `SidebarItem` for "Preview" with appropriate icon

### 21. New: Playwright E2E test

End-to-end test verifying:
- Navigate to `/preview`
- Paste valid JSON
- Verify graph renders with preview styling
- Test share URL generation
- Test DOT export

## Implementation Order

| Step | Files | Depends On | User Story |
|------|-------|-----------|-----------|
| 1 | `graphImport.ts`, `graphImport.test.ts` | — | US1 |
| 2 | `AppGraph.tsx` (preview prop), `ResourceNode.tsx` (preview styling) | — | US2 |
| 3 | `PreviewBanner.tsx` | — | US2 |
| 4 | `GraphImportPanel.tsx`, stories, tests | Step 1 | US1 |
| 5 | `PreviewPage.tsx`, routes, plugin, App.tsx, Root.tsx | Steps 1-4 | US1+US2 |
| 6 | `dotExport.ts`, `dotExport.test.ts` | Step 1 | US4 |
| 7 | `GraphActions.tsx` | Steps 5, 6 | US4 |
| 8 | `shareableUrl.ts`, `shareableUrl.test.ts` | Step 1 | US3 |
| 9 | Integrate share URL into PreviewPage | Steps 5, 8 | US3 |
| 10 | Playwright E2E tests | Steps 1-9 | All |

## Package Dependencies

New packages to add to `packages/rad-components/package.json`:

| Package | Purpose |
|---------|---------|
| `pako` | Deflate compression for shareable URLs |
| `@types/pako` | TypeScript types for pako |

No other new dependencies needed — React Flow, Dagre, Material UI, and React are already present.
