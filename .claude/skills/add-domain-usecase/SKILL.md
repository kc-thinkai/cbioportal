---
name: add-domain-usecase
description: >-
  Add domain-layer use cases, repository interfaces, and matching tests
  on the new persistence stack. Use when adding domain features, ClickHouse
  repositories, or following the cBioPortal golden path.
---

# Add domain use case

New features use `src/main/java/org/cbioportal/domain/`, not the legacy service layer.

## Pattern

1. Use case injects a domain repository interface and delegates persistence.
2. Unit-test the use case with Mockito.
3. Integration-test ClickHouse implementations against a real DB (`AbstractClickhouseIntegrationTest`).

## Canonical files

| Layer | Example |
|-------|---------|
| Use case | `src/main/java/org/cbioportal/domain/cancerstudy/usecase/GetCancerStudyMetadataUseCase.java` |
| Unit test | `src/test/java/org/cbioportal/cancerstudy/usecase/GetCancerStudyMetadataUseCaseTest.java` |
| Repo IT | `src/integration/java/org/cbioportal/infrastructure/repository/clickhouse/cancerstudy/ClickhouseCancerStudyRepositoryIntegrationTest.java` |

Place new domain tests under `src/test/java/org/cbioportal/domain/` when adding new packages.

## Build

- Compile: `mvn install -DskipTests`
- Tests: `mvn integration-test`
- Target branch: `master` (v7)
