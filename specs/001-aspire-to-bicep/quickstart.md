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
- **Warnings**: Check for `// Unsupported:` comments marking resources that need manual attention (including `parameter.v0` resources that require manual Bicep parameter declarations).
- **Build warnings**: If you see comments about build configurations, ensure those images are pre-built and pushed.

## Step 4: Deploy with Radius

```bash
rad deploy app.bicep
```

If you need to supply parameters (e.g., for secrets that were not automatically converted), pass them explicitly:

```bash
rad deploy app.bicep --parameters cachePassword=mysecretpassword
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
#     ✓ frontend (container.v1) → Radius.Compute/containers
#   Warnings:
#     ⚠ app: has build configuration
#     ⚠ frontend: has build configuration
#     ⚠ cache-password: unsupported (parameter.v0)
#     ⚠ cache-password-uri-encoded: unsupported (annotated.string)
#   Generated: app.bicep (3 containers, 1 gateway, 2 skipped)

# Build and push your container images (if not already done)
docker build -t myregistry/app:latest ./app
docker push myregistry/app:latest

# Update app.bicep with your actual image references and add any needed parameters, then deploy
rad deploy app.bicep
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
#     ✓ frontend (container.v1) → Radius.Compute/containers
#   Warnings:
#     ⚠ docker-hub: manifest error — This resource does not support generation in the manifest.
#     ⚠ app: has build configuration
#     ⚠ frontend: has build configuration
#     ⚠ cache-password: unsupported (parameter.v0)
#     ⚠ cache-password-uri-encoded: unsupported (annotated.string)
#   Generated: app.bicep (3 containers, 1 gateway, 3 skipped)

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
