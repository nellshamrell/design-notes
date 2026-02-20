# Specification Quality Checklist: Aspire Manifest to Bicep Conversion

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-02-19
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [x] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- All items pass validation. Spec is ready for `/speckit.clarify` or `/speckit.plan`.
- The spec references specific Bicep resource type API versions in the Assumptions section for context, but does not prescribe which version to use — this is deferred to implementation.
- FR-002 mentions two possible resource type names (`Applications.Core/containers` and `Radius.Compute/containers`) because the project is transitioning naming conventions; the spec does not mandate which to use.
- FR-018 added (2026-02-20) to handle Aspire manifest resource entries with `error` fields (no `type`). This covers the `aspire-manifest-invalid-manifest-field.json` scenario where resources like `docker-hub` have a manifest-publisher error instead of a resource type. The conversion must still succeed for all other valid resources.
- FR-019 added (2026-02-20) to exclude `container.v1` resources with `build.buildOnly: true`. These are build-time-only artifacts (e.g., the `frontend` resource in `aspire-manifest-invalid-manifest-field.json`) that produce static files consumed by other containers via `containerFiles`. They are not runtime containers and must not be converted to Radius resources. A resource exclusion priority order is documented in the spec: error field → buildOnly → unsupported type → normal mapping.
- FR-020 added (2026-02-20) to handle `parameter.v0` resources with `secret: true` inputs. These map to `@secure() param` declarations in Bicep (e.g., `cache-password` with secret input → `@secure() param cache_password string`). Non-secret parameters are treated as unsupported and logged with a warning.
- FR-021 added (2026-02-20) to handle `annotated.string` resources with `filter: "uri"`. These map to `var name = uriComponent(paramRef)` declarations in Bicep (e.g., `cache-password-uri-encoded` → `var cache_password_uri_encoded = uriComponent(cache_password)`). The variable provides a URI-encoded version of the referenced parameter for use in connection strings.
- FR-004 updated (2026-02-20) to reflect full expression resolution rules: binding host/port references resolve to string literals (not resource references), `parameter.v0` references resolve to bare Bicep parameter names, `annotated.string` references resolve to variable names, connection strings are fully expanded recursively, and composite expressions use Bicep string interpolation.
- FR-007 updated (2026-02-20) to require only `extension radius` (no `extension containers` or other extensions). All Radius resource types are available through this single extension.
