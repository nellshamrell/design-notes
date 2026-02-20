# Quickstart: Aspire Manifest to Bicep Conversion

**Feature**: 001-aspire-to-bicep
**Date**: 2026-02-19

## Prerequisites

- The `rad` CLI is installed and configured with a Radius environment.
- You have an Aspire manifest JSON file (generated via `dotnet run --publisher manifest` or similar).
- Container images referenced in the manifest are built and pushed to an accessible registry.

## Step 1: Generate the Aspire manifest (if you haven't already)

```bash
# From your .NET Aspire project directory:
dotnet run --project MyAspireApp.AppHost --publisher manifest --output-path aspire-manifest.json
```

## Step 2: Convert the manifest to a Radius Bicep file

```bash
rad aspire convert aspire-manifest.json
```

This produces `app.bicep` in the current directory.

### Specify a custom output path

```bash
rad aspire convert aspire-manifest.json --output my-app.bicep
```

### Overwrite an existing file

```bash
rad aspire convert aspire-manifest.json --force
```

### Set a custom application name

```bash
rad aspire convert aspire-manifest.json --application my-aspire-app
```

## Step 3: Review the output

Open `app.bicep` and review:

- **Container resources**: Verify images, ports, and environment variables are correct.
- **Secure parameters**: Secret parameters (e.g., `cache_password`) are generated as `@secure()` Bicep parameters. You will supply these at deploy time.
- **Variables**: URI-encoded values (e.g., `cache_password_uri_encoded`) are generated as Bicep variables using `uriComponent()`.
- **Warnings**: Check for `// Unsupported:` comments marking resources that need manual attention.
- **Build warnings**: If you see comments about build configurations, ensure those images are pre-built and pushed.

## Step 4: Deploy with Radius

```bash
rad deploy app.bicep
```

If the generated file includes `@secure()` parameters (e.g., for secrets like `cache_password`), supply them at deploy time:

```bash
rad deploy app.bicep --parameters cache_password=mysecretpassword
```

## Example: Converting the sample manifest

Using the provided `aspire-manifest.json` (Redis cache + Python app + Node.js frontend):

```bash
# Convert
rad aspire convert aspire-manifest.json

# Review summary output:
#   Converted resources:
#     ✓ cache (container.v0) → Radius.Compute/containers
#     ✓ app (container.v1) → Radius.Compute/containers
#     ✓ cache-password (parameter.v0, secret) → @secure() param cache_password
#     ✓ cache-password-uri-encoded (annotated.string, uri) → var cache_password_uri_encoded
#   Warnings:
#     ⚠ app: has build configuration
#     ⚠ frontend: skipped — build-only artifact (build.buildOnly: true)
#     ⚠ cache-password-uri-encoded: unsupported (annotated.string) — best-effort uriComponent() mapping
#   Generated: app.bicep (2 containers, 1 gateway, 1 param, 1 var, 1 skipped)

# Build and push your container images (if not already done)
docker build -t myregistry/app:latest ./app
docker push myregistry/app:latest

# Update app.bicep with your actual image references, then deploy
rad deploy app.bicep --parameters cache_password=mysecretpassword
```

## Example: Converting a manifest with errored resources

Some Aspire manifests include resource entries that the manifest publisher could not generate (e.g., custom Docker registries). These entries have an `error` field instead of a `type` field. The conversion command handles these gracefully:

```bash
# Convert a manifest that includes an errored resource entry
rad aspire convert aspire-manifest-invalid-manifest-field.json

# Review summary output:
#   Converted resources:
#     ✓ cache (container.v0) → Radius.Compute/containers
#     ✓ app (container.v1) → Radius.Compute/containers
#     ✓ cache-password (parameter.v0, secret) → @secure() param cache_password
#     ✓ cache-password-uri-encoded (annotated.string, uri) → var cache_password_uri_encoded
#   Warnings:
#     ⚠ docker-hub: manifest error — This resource does not support generation in the manifest.
#     ⚠ app: has build configuration
#     ⚠ frontend: skipped — build-only artifact (build.buildOnly: true)
#     ⚠ cache-password-uri-encoded: unsupported (annotated.string) — best-effort uriComponent() mapping
#   Generated: app.bicep (2 containers, 1 gateway, 1 param, 1 var, 2 skipped)

rad deploy app.bicep
```

The errored resource (`docker-hub`) is skipped with a warning, and the output Bicep file includes a comment noting the skipped resource. All other resources convert normally.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Error: file not found` | Manifest path is incorrect | Verify the path to your JSON file |
| `Error: invalid JSON` | Manifest file has syntax errors | Validate with `jq . aspire-manifest.json` |
| `Error: output file already exists` | Target file exists | Use `--force` to overwrite, or `--output` for a different path |
| Bicep compilation errors | Unsupported resource was mapped incorrectly | Check `// Unsupported:` comments in the Bicep file; edit manually |
| `⚠ manifest error` warning | Aspire manifest publisher could not generate a resource | The resource is skipped; no action needed unless you expected it to be converted |
| Missing environment variables | Expression references couldn't be fully resolved | Check warnings; some inter-resource references may need manual Bicep edits |
