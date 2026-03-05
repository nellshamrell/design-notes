# Contract: Shareable URL Encoding

**Date**: 2026-03-05
**Feature**: [spec.md](../spec.md) | [data-model.md](../data-model.md)

## Overview

This contract defines the interface for encoding and decoding application graph data in shareable URLs. All data is stored in the URL hash fragment — no server-side storage is involved.

## URL Format

```
https://{dashboard-host}/preview#graph={encoded-data}
```

- The hash fragment (`#graph=...`) is used because:
  - Not sent to the server (client-side only)
  - Not subject to query string length limits
  - Survives page reloads
- Practical browser limit for hash fragments: ~64KB (varies by browser)

## Encoding Contract

### Function: `encodeGraphUrl`

```typescript
/**
 * Encodes ApplicationGraphResponse JSON into a shareable URL.
 * Uses deflate compression + Base64 URL-safe encoding.
 *
 * @param response - The validated ApplicationGraphResponse
 * @param baseUrl - The dashboard's base URL (e.g., window.location.origin + '/preview')
 * @returns Encoding result with URL or error
 */
function encodeGraphUrl(
  response: ApplicationGraphResponse,
  baseUrl: string
): EncodeResult;

interface EncodeResult {
  success: boolean;
  url?: string;       // Present when success === true
  error?: string;     // Present when success === false
}
```

### Encoding steps

1. Serialize `ApplicationGraphResponse` to JSON string (minified, no whitespace)
2. Compress with deflate (pako or equivalent)
3. Encode compressed bytes as Base64 URL-safe string (`+` → `-`, `/` → `_`, no padding `=`)
4. Construct URL: `{baseUrl}#graph={base64}`
5. If resulting URL length > 64,000 characters, return error: `"Graph data is too large to share via URL. Export the JSON file instead."`

## Decoding Contract

### Function: `decodeGraphUrl`

```typescript
/**
 * Extracts and decodes ApplicationGraphResponse from a URL hash fragment.
 *
 * @param hash - The URL hash string (e.g., "#graph=...")
 * @returns Decode result with validated data or error
 */
function decodeGraphUrl(
  hash: string
): ValidationResult<ApplicationGraphResponse>;
```

### Decoding steps

1. Extract `graph=` parameter from hash fragment
2. If no `graph=` parameter, return `{ success: false, errors: ["No graph data found in URL"] }`
3. Decode Base64 URL-safe string to bytes
4. Decompress with inflate (pako or equivalent)
5. Parse JSON string
6. Validate with `validateApplicationGraphResponse`
7. Return validation result

### Error handling

| Condition | Error message |
|-----------|--------------|
| No `graph=` in hash | `"No graph data found in URL"` |
| Invalid Base64 | `"Unable to decode shared link: invalid encoding"` |
| Decompression failure | `"Unable to decode shared link: data appears corrupted"` |
| Invalid JSON after decode | `"Unable to decode shared link: invalid data format"` |
| Schema validation failure | Delegate to `validateApplicationGraphResponse` errors |

## Clipboard Contract

### Function: `copyShareUrl`

```typescript
/**
 * Generates the shareable URL and copies it to the clipboard.
 * Updates the browser URL hash to match.
 *
 * @param response - The current ApplicationGraphResponse
 * @returns Promise resolving to the encode result
 */
function copyShareUrl(
  response: ApplicationGraphResponse
): Promise<EncodeResult>;
```

### Behavior

1. Call `encodeGraphUrl` with current response and `window.location.origin + '/preview'`
2. If successful, update `window.location.hash` to the encoded hash
3. Copy the full URL to clipboard via `navigator.clipboard.writeText`
4. Return the encode result
