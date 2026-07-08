# AI Agent Guidelines for cBioPortal

This file contains rules and conventions that AI coding agents (Claude Code, Copilot, Cursor, etc.) must follow when contributing to this project.

## Project Context

Long-lived project documentation and code organization live in several places:

| Location | Contents |
|----------|----------|
| **`AGENTS.md`** (this file) | Agent conventions, security rules, and golden-path examples |
| **[`docs/`](docs/)** | Full project documentation — start at [`docs/README.md`](docs/README.md). Key developer sections: [`docs/development/`](docs/development/) (backend organization, feature development, security), [`docs/deployment/`](docs/deployment/), and [`docs/ai-integrations/`](docs/ai-integrations/) |
| **[`README.md`](README.md)** | Backend setup, branching/release strategy, and the **[Testing Overview](README.md#testing-overview)** (unit, integration, and E2E layers, directory layout, env vars, Maven profiles) |
| **Module READMEs** | Focused context for specific areas: [`application/security/README.md`](src/main/java/org/cbioportal/application/security/README.md), [`application/file/README.md`](src/main/java/org/cbioportal/application/file/README.md), [`src/e2e/js/README.md`](src/e2e/js/README.md) |

### Domain package layout

New features use the domain/infrastructure stack (not the legacy `service`/`persistence-mybatis` layers):

| Path | Purpose |
|------|---------|
| `src/main/java/org/cbioportal/domain/<context>/` | Bounded contexts (e.g. `cancerstudy/`, `sample/`, `patient/`, `mutation/`, `clinical_data/`, `genomic_data/`) |
| `.../domain/<context>/usecase/` | Use-case classes (business logic entry points) |
| `.../domain/<context>/repository/` | Domain repository interfaces |
| `src/main/java/org/cbioportal/infrastructure/repository/` | Repository implementations (e.g. ClickHouse) |
| `src/test/java/org/cbioportal/domain/` | Use-case unit tests (Mockito) |
| `src/integration/java/org/cbioportal/infrastructure/repository/` | Repository integration tests against real databases |

For the legacy Maven module structure (web, service, persistence), see [`docs/development/Backend-Code-Organization.md`](docs/development/Backend-Code-Organization.md).

## Build & Test

- Build: `mvn install -DskipTests`
- Run all tests: `mvn integration-test`
- All new development targets the `master` branch (v7)

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

## Canonical Example (Golden Path)

When adding domain-layer features, follow this use-case + test pattern:

| Layer | Example |
|-------|---------|
| Use case | `src/main/java/org/cbioportal/domain/cancerstudy/usecase/GetCancerStudyMetadataUseCase.java` |
| Use case unit test | `src/test/java/org/cbioportal/cancerstudy/usecase/GetCancerStudyMetadataUseCaseTest.java` |
| Repository integration test | `src/integration/java/org/cbioportal/infrastructure/repository/clickhouse/cancerstudy/ClickhouseCancerStudyRepositoryIntegrationTest.java` |

The use case injects a domain repository interface, delegates persistence to it, and is covered by a Mockito unit test. The ClickHouse repository implementation is verified against a real database in `src/integration/` (extends `AbstractClickhouseIntegrationTest`). New domain code belongs under `src/main/java/org/cbioportal/domain/` with matching tests under `src/test/java/org/cbioportal/domain/`.
