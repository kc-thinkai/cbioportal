# AI Agent Guidelines for cBioPortal

This file contains rules and conventions that AI coding agents (Claude Code, Copilot, Cursor, etc.) must follow when contributing to this project.

## Build & Test

Use the Maven wrapper (`./mvnw`) for reproducible builds; a system Maven install is optional.

- Build: `./mvnw install -DskipTests`
- Unit tests: `./mvnw test`
- Integration tests: `./mvnw verify -Pintegration-test` (requires env vars; see `.env.example`)
- Lint/format: `./mvnw spotless:check` / `./mvnw spotless:apply`
- All new development targets the `master` branch (v7)

## Configuration

Local settings are split between environment variables and Spring property files.

- **Environment variables:** copy `.env.example` to `.env` (or `dev/.env` for Docker Compose) and set required values. Integration and E2E tests read `TEST_DB_*` and `CBIOPORTAL_URL` from the environment.
- **Application properties:** copy `src/main/resources/application.properties.EXAMPLE` → `application.properties` and `src/main/resources/security.properties.EXAMPLE` → `security.properties` (both are gitignored). Database, auth, and portal skin settings live here.

### Spring Boot profiles (`spring.profiles.active`)

| Profile | Purpose |
|---------|---------|
| `clickhouse` | v7 default backend; ClickHouse column-store APIs (CI uses `--spring.profiles.active=clickhouse`) |
| `dbcp` | Legacy study importer only (`cbioportal-core`; not used by the web app) |

Pass at runtime, e.g. `java -jar target/cbioportal-exec.jar --spring.profiles.active=clickhouse`.

### Maven profiles (`-P`)

| Profile | Purpose | Command |
|---------|---------|---------|
| *(default)* | Unit tests only | `./mvnw test` |
| `integration-test` | DB / Spring integration tests | `./mvnw verify -Pintegration-test` |
| `e2e-test` | JavaScript API E2E tests | `./mvnw verify -Pe2e-test` |

## Endpoint Authorization (Security-Critical)

Every REST controller endpoint that accesses study-specific data **must** have a `@PreAuthorize` annotation. Forgetting this allows unauthorized data access.

### Patterns

- **GET endpoints with `studyId`:**
  ```java
  @PreAuthorize("hasPermission(#studyId, 'CancerStudyId', T(org.cbioportal.legacy.utils.security.AccessLevel).READ)")
  ```

- **POST fetch endpoints with filters (study collection):**
  ```java
  @PreAuthorize("hasPermission(#involvedCancerStudies, 'Collection<CancerStudyId>', T(org.cbioportal.legacy.utils.security.AccessLevel).READ)")
  ```
  These also require the `InvolvedCancerStudyExtractorInterceptor` (in `WebAppConfig`) to handle the endpoint path.

### Exceptions (no @PreAuthorize needed)

Controllers serving only public/reference data do not need authorization. These include:
- `CancerTypeController` — public reference data
- `GeneController` — public reference data
- `GenePanelController` — public reference data
- `GenesetController` — public reference data
- `ReferenceGenomeGeneController` — public reference data
- `InfoController` — server metadata
- `ServerStatusController` — health check
- `CacheController` / `CacheStatsController` — operational
- `IndexPageController` / `LoginPageController` — UI pages
- `PublicVirtualStudiesController` — explicitly public

The following controllers have known authorization gaps that are tracked as TODOs:
- `MutationCountController` — returns aggregate mutation counts across all studies with no study IDs in the request or response; `@PostFilter` is not applicable; authorization approach needs further investigation
- `MskEntityTranslationController` — accesses study-specific data but lacks `@PreAuthorize`; currently allowed in the ArchUnit test to avoid breaking the build
- `GenericAssayController` — accesses study-specific data; `@PreAuthorize` was removed for performance reasons; currently allowed in the ArchUnit test to avoid breaking the build
- `ColumnStoreGenericAssayController` — accesses study-specific data; `@PreAuthorize` was removed for performance reasons; currently allowed in the ArchUnit test to avoid breaking the build
- `ColumnStoreStudyController` — serves study data without per-study authorization; currently allowed in the ArchUnit test to avoid breaking the build

Any new exception must be documented here with a justification.

### Enforcement

An ArchUnit test (`EndpointAuthorizationArchTest`) verifies that all `@RequestMapping` methods in `@RestController` classes have `@PreAuthorize`, unless the controller or method is in a documented exceptions list. If you add a new endpoint, the test will fail unless you either:
1. Add `@PreAuthorize` to the method, OR
2. Add the controller to `AUTHORIZED_EXCEPTIONS` (or the method to `METHOD_EXCEPTIONS`) with a justification comment

## Code Conventions

- Follow existing patterns in the codebase when adding new endpoints or services
- New features should use the new persistence stack (domain/infrastructure layers), not the legacy service layer
- PRs should include test coverage for new functionality
