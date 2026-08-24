---
name: add-authorized-endpoint
description: >-
  Add or update REST endpoints with required @PreAuthorize study access
  checks. Use when adding controllers, RequestMapping methods, fetch
  filters, or when EndpointAuthorizationArchTest fails.
---

# Add authorized endpoint

Every REST method that accesses study-specific data must have `@PreAuthorize`.

## GET with studyId

```java
@PreAuthorize("hasPermission(#studyId, 'CancerStudyId', T(org.cbioportal.legacy.utils.security.AccessLevel).READ)")
```

## POST fetch with study collection

```java
@PreAuthorize("hasPermission(#involvedCancerStudies, 'Collection<CancerStudyId>', T(org.cbioportal.legacy.utils.security.AccessLevel).READ)")
```

Register the path in `InvolvedCancerStudyExtractorInterceptor` (`WebAppConfig`).

## Public/reference controllers

Do not add `@PreAuthorize` for controllers listed as public in `AGENTS.md` (gene, cancer type, info, health, login, etc.).

## New exceptions

If a method cannot use `@PreAuthorize`, document justification in `AGENTS.md` and add the controller/method to `AUTHORIZED_EXCEPTIONS` or `METHOD_EXCEPTIONS` in `EndpointAuthorizationArchTest`.
