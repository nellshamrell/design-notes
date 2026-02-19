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
- **Secure parameters**: Note any `@secure()` parameters — you'll provide values at deploy time.
- **Warnings**: Check for `// Unsupported:` comments marking resources that need manual attention.
- **Build warnings**: If you see comments about build configurations, ensure those images are pre-built and pushed.

## Step 4: Deploy with Radius

```bash
rad deploy app.bicep
```

If the Bicep file has secure parameters, you'll be prompted for values or can pass them:

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
#     ✓ cache-password (parameter.v0) → @secure() parameter
#   Warnings:
#     ⚠ app: has build configuration
#     ⚠ frontend: has build configuration
#     ⚠ cache-password-uri-encoded: unsupported (annotated.string)
#   Generated: app.bicep (3 containers, 1 parameter, 1 gateway, 1 skipped)

# Build and push your container images (if not already done)
docker build -t myregistry/app:latest ./app
docker push myregistry/app:latest

# Update app.bicep with your actual image references, then deploy
rad deploy app.bicep --parameters cachePassword=mypassword
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Error: file not found` | Manifest path is incorrect | Verify the path to your JSON file |
| `Error: invalid JSON` | Manifest file has syntax errors | Validate with `jq . aspire-manifest.json` |
| `Error: output file already exists` | Target file exists | Use `--force` to overwrite, or `--output` for a different path |
| Bicep compilation errors | Unsupported resource was mapped incorrectly | Check `// Unsupported:` comments in the Bicep file; edit manually |
| Missing environment variables | Expression references couldn't be fully resolved | Check warnings; some inter-resource references may need manual Bicep edits |
