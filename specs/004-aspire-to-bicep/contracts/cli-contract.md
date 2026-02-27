# CLI Contract: `rad bicep generate --from-aspire`

**Branch**: `004-aspire-to-bicep` | **Date**: 2026-02-23

## Command Syntax

```
rad bicep generate --from-aspire <path> [flags]
```

## Description

Generate a Radius-deployable app.bicep file from Aspire-generated infrastructure artifacts (azd infra synth output).

This command reads the `azd infra synth` output for a .NET Aspire application — including per-service YAML templates (`.tmpl.yaml`) and solution-level Bicep files — and produces a Radius-compatible app.bicep file along with a mapping report documenting the conversion.

## Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| (none) | — | No positional arguments |

## Flags

| Flag | Short | Type | Default | Required | Description |
|------|-------|------|---------|----------|-------------|
| `--from-aspire` | — | `string` | — | Yes | Path to the Aspire application root directory containing `azd infra synth` output (solution-level `infra/` and AppHost-level `<AppHost>/infra/*.tmpl.yaml` files) |
| `--output` | `-o` | `string` | `./app.bicep` | No | Path for the generated app.bicep output file |
| `--app-name` | — | `string` | (derived from Aspire project) | No | Name for the Radius application resource |
| `--report` | — | `string` | `./mapping-report.md` | No | Path for the companion mapping report Markdown file |
| `--quiet` | `-q` | `bool` | `false` | No | Suppress console mapping report output (file still generated) |
| `--deterministic` | — | `bool` | `false` | No | Replace timestamps in output with a fixed sentinel value for idempotency verification in CI |

## Input Contract

### Directory Structure

The `--from-aspire` path must point to an Aspire application root directory with the following structure (as verified against the reference application at `./example-aspire-app`):

```
<path>/
├── infra/
│   ├── main.bicep              # Required: subscription-scoped orchestrator with parameters
│   ├── main.parameters.json    # Optional: parameter values (environmentName, passwords)
│   └── resources.bicep         # Optional: Azure infra (managed identity, ACR, container env)
├── <AppHost>/                  # Required: AppHost project directory (auto-detected by *.csproj containing Aspire.AppHost)
│   ├── AppHost.cs              # Optional: topology definition (WithReference, WaitFor)
│   └── infra/
│       ├── <service>.tmpl.yaml # Required: at least one per-service YAML template
│       └── ...                 # Additional service/dependency YAML templates
└── ...                         # Other project directories (ApiService, Web, ServiceDefaults)
```

### AppHost Detection

The CLI auto-detects the AppHost project directory by scanning for a `.csproj` file that references `Aspire.AppHost` or by finding a directory containing an `infra/` subdirectory with `.tmpl.yaml` files. If multiple AppHost directories are found, the command fails with an FR-011 error.

### Required: Per-Service YAML Templates

Each `.tmpl.yaml` file in `<AppHost>/infra/` is expected to define an Azure Container App with:
- `tags.aspire-resource-name` identifying the Aspire resource name
- `properties.configuration.ingress` block with `targetPort`, `external`, `transport`
- `properties.template.containers[]` array with `image`, `name`, `env[]`
- Optional: `properties.configuration.secrets[]` for connection strings
- Go template expressions (`{{ .Image }}`, `{{ .Env.* }}`, `{{ securedParameter "..." }}`, `{{ targetPortOrDefault N }}`) are handled by stripping/replacing before YAML parsing

**Reference application templates** (`./example-aspire-app/AspireApp.AppHost/infra/`):
- `apiservice.tmpl.yaml` — internal HTTP service (port 8080), connects to sqlserver via `ConnectionStrings__weatherdb`
- `webfrontend.tmpl.yaml` — external HTTP service (port 8080), connects to apiservice via `services__apiservice__http__0` and to cache via `ConnectionStrings__cache`
- `cache.tmpl.yaml` — Redis container (port 6379/tcp), `REDIS_PASSWORD` secret
- `sqlserver.tmpl.yaml` — SQL Server container (port 1433/tcp), `MSSQL_SA_PASSWORD` secret

### Optional: Solution-Level Bicep

The solution-level `infra/main.bicep` is read for:
- `param environmentName` — used to derive the application name
- `@secure() param` declarations — identifies secured parameters (e.g., `cache_password`, `sqlserver_password`)
- Module structure — provides context about Azure infrastructure topology

### Service vs. Dependency Classification

Services and dependencies are classified by heuristics applied to `.tmpl.yaml` content:
- **Service**: Has `ingress.transport: http` and corresponds to an `AddProject<>()` call in AppHost.cs
- **Dependency (Redis)**: Has `ingress.transport: tcp` + `targetPort: 6379` or `redis` in name/image
- **Dependency (SQL Server)**: Has `ingress.transport: tcp` + `targetPort: 1433` or `sql`/`mssql`/`sqlserver` in name/image
- **Other**: Unknown dependencies are classified as placeholders

## Output Contract

### Generated File: `app.bicep`

The output file follows this structure (based on the reference application `./example-aspire-app`):

```bicep
// Generated by: rad bicep generate --from-aspire
// Source: <input-directory-path>
// Date: <ISO-8601-timestamp>

extension radius

@description('The ID of your Radius Environment. Set automatically by the rad CLI.')
param environment string

@description('The name of the Radius Application.')
param applicationName string = '<app-name>'

@description('Container image for <service-name>.')
param <serviceName>Image string = 'IMAGE_PLACEHOLDER'
// ... one param per service

resource app 'Applications.Core/applications@2023-10-01-preview' = {
  name: applicationName
  properties: {
    environment: environment
  }
}

resource <serviceName> 'Applications.Core/containers@2023-10-01-preview' = {
  name: '<service-name>'
  properties: {
    application: app.id
    environment: environment
    container: {
      image: <serviceName>Image
      ports: {
        <portName>: {
          containerPort: <port>
          protocol: '<protocol>'
        }
      }
      env: {
        // Environment variables use { value: '<literal>' } syntax.
        // For dependency-backed env vars, Bicep resource expressions are used:
        //   ConnectionStrings__<dep>: { value: <dep>.listSecrets().connectionString }
        //   <DEP>_HOST: { value: <dep>.properties.host }
        //   <DEP>_PORT: { value: string(<dep>.properties.port) }
        //   <DEP>_PASSWORD: { value: <dep>.listSecrets().password }
      }
    }
    connections: {
      <connectionName>: {
        source: <targetResource>.id
      }
    }
  }
}

// ... additional containers

resource <depName> 'Applications.Datastores/redisCaches@2023-10-01-preview' = {
  name: '<dep-name>'
  properties: {
    application: app.id
    environment: environment
  }
}

resource <depName> 'Applications.Datastores/sqlDatabases@2023-10-01-preview' = {
  name: '<dep-name>'
  properties: {
    application: app.id
    environment: environment
  }
}

// PLACEHOLDER: <unsupported-dep-name> (<type>)
// No Radius Portable Resource equivalent exists for <type>.
// Consider using a Recipe or manual resource configuration.
// Source: <AppHost>/infra/<dep-name>.tmpl.yaml
```

**Reference application expected output** (`./example-aspire-app`):
- Application: `example-aspire-app`
- Containers: `apiservice` (port 8080/http, connection to `sqlserver`), `webfrontend` (port 8080/http, external, connections to `apiservice` + `cache`)
- Dependencies: `cache` (`Applications.Datastores/redisCaches`, Recipe-backed), `sqlserver` (`Applications.Datastores/sqlDatabases`, Recipe-backed)
- Image defaults use `IMAGE_PLACEHOLDER`, overridable at deploy time via `rad deploy --parameters`
- Environment variables for dependency connections use Bicep resource expressions (e.g., `sqlserver.listSecrets().connectionString`, `cache.properties.host`) rather than raw string values

### Generated File: `mapping-report.md`

```markdown
# Mapping Report: Aspire-to-Bicep Conversion

**Source**: <input-directory-path>
**Generated**: <ISO-8601-timestamp>
**Output**: <output-file-path>

## Resource Summary

| # | Radius Resource | Type | Source File | Source Resource |
|---|----------------|------|-------------|----------------|
| 1 | app | Applications.Core/applications | main.bicep | (derived) |
| 2 | apiservice | Applications.Core/containers | AspireApp.AppHost/infra/apiservice.tmpl.yaml | Container App (YAML template) |
| ... | ... | ... | ... | ... |
## Field-Level Mapping

### <resource-name> (Applications.Core/containers)

| Target Field | Value | Source File | Source Field | Status |
|-------------|-------|-------------|-------------|--------|
| name | apiservice | apiservice.tmpl.yaml | tags.aspire-resource-name | Mapped |
| container.image | apiserviceImage (param) | apiservice.tmpl.yaml | template.containers[0].image | Mapped |
| container.ports.http.containerPort | 8080 | apiservice.tmpl.yaml | configuration.ingress.targetPort | Mapped |
| ... | ... | ... | ... | ... |

## Gaps

| Resource | Field | Expected Source | Status | Action Required |
|----------|-------|----------------|--------|-----------------|
| ... | ... | ... | Missing | ... |

## Assumptions

| Resource | Field | Default Value | Rationale |
|----------|-------|---------------|-----------|
| ... | ... | ... | ... |
```

### Console Output

Success (reference application `./example-aspire-app`):
```
Converting Aspire artifacts from: ./example-aspire-app
  Found AppHost: AspireApp.AppHost/infra/ (4 templates)
  Found 2 services: apiservice, webfrontend
  Found 2 dependencies: cache (Redis), sqlserver (SQL Server)

Generating app.bicep...
  ✓ Application: example-aspire-app
  ✓ Container: apiservice (image param: apiserviceImage, default: IMAGE_PLACEHOLDER)
  ✓ Container: webfrontend (image param: webfrontendImage, default: IMAGE_PLACEHOLDER)
  ✓ Env vars: Bicep resource expressions for dependency connections (e.g., sqlserver.listSecrets())
  ✓ Dependency: cache (Applications.Datastores/redisCaches, Recipe-backed)
  ✓ Dependency: sqlserver (Applications.Datastores/sqlDatabases, Recipe-backed)
  ✓ Connection: webfrontend → apiservice
  ✓ Connection: webfrontend → cache
  ✓ Connection: apiservice → sqlserver

Output written to: ./app.bicep
Mapping report written to: ./mapping-report.md

Gaps found: 0
Assumptions made: 3 (see mapping report for details)
```

Error (no AppHost infra directory):
```
Error: Input directory './some-dir' does not contain Aspire infrastructure artifacts.
Expected: An Aspire application directory with <AppHost>/infra/*.tmpl.yaml files.
Run 'azd infra synth' in your Aspire project to generate the required artifacts.
```

Error (multiple projects):
```
Error: Multiple Aspire projects detected in './infra'.
This PoC supports a single Aspire project only. Please provide a directory with one main.bicep file.
```

## Determinism (FR-012)

The command guarantees byte-for-byte identical output when run on identical input artifacts:

- All resources, parameters, and mapping entries are emitted in lexicographic order by name
- Map-typed fields (environment variables) are iterated in sorted key order
- File discovery uses lexicographic path ordering
- When `--deterministic` is set, the `Date:` comment in `app.bicep` and `Generated:` field in `mapping-report.md` use a fixed sentinel (`DETERMINISTIC`) instead of the current timestamp

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success — app.bicep and mapping-report.md generated |
| 1 | Error — invalid input, missing files, or conversion failure |

## Examples

```bash
# Basic usage — convert the reference Aspire application
rad bicep generate --from-aspire ./example-aspire-app

# Specify output paths
rad bicep generate --from-aspire ./example-aspire-app --output ./deploy/app.bicep --report ./deploy/mapping-report.md

# Override application name
rad bicep generate --from-aspire ./example-aspire-app --app-name my-app

# Quiet mode (suppress console report)
rad bicep generate --from-aspire ./example-aspire-app --quiet

# Full workflow example
cd example-aspire-app
azd infra synth                                                                    # Generate Aspire artifacts (if not already present)
rad bicep generate --from-aspire .                                                 # Convert to Radius
rad deploy app.bicep --parameters apiserviceImage=myregistry/apiservice:latest \    # Deploy with image overrides
  webfrontendImage=myregistry/webfrontend:latest
```
