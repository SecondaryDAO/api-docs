# PURPOSE — api-docs

**Updated:** 2026-10-16

## Why this exists

Give API consumers an accurate contract for SecondaryDAO's supported programmatic interfaces, authentication, data, transaction prerequisites, and failure responses.

## Relationship to the platform

The [SecondaryDAO root charter](https://github.com/SecondaryDAO/SD/blob/local2/PURPOSE.md) defines platform purpose, entity roles, provisional offering direction, and shared launch criteria. Read it alongside current approved decisions. Root-charter corrections are tracked in [SD #124](https://github.com/SecondaryDAO/SD/pull/124) for local2 and [SD #123](https://github.com/SecondaryDAO/SD/pull/123) for development; their PR state determines whether the corrections have landed.

This file defines this repository's responsibility and acceptance criteria. Do not copy legal thresholds or financial mechanics here; follow the approved decision and its canonical implementation, and flag discrepancies rather than silently choosing one.

## Responsibilities

- Keep human-readable guidance, rendered documentation, and OpenAPI request/response definitions aligned with supported backend routes.
- Document authentication, authorization, eligibility, units, pagination, errors, and transaction state accurately.
- Identify removed or unsupported interfaces clearly and keep examples free of credentials and unsupported completion claims.

## Boundaries

Documentation does not create an endpoint, legal permission, or a second set of financial and compliance rules. A historical example is not evidence that its route remains supported. Verify against the backend before publishing behavior changes.

## Success criteria

A consumer can follow the documented authentication and supported workflows against the identified release, interpret pending versus confirmed results, and receive the documented errors for rejected requests.

## How to use this file

For a specific request, perform the requested work. For an open-ended session, identify the smallest verified gap between this repository's responsibility and the integrated first-property lifecycle. Report coded, tested, merged, deployed, and production-verified states separately. This purpose file is a target and boundary, not evidence that its success criteria have already passed.
