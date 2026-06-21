# ADR-001: A single platform home repo (`enpicie-platform`)

## Status
Accepted

## Date
2026-06-21

## Context
Shared platform knowledge in GitHub is spread thin across the `Infrastructure-Documentation` repo, per-repo READMEs, and per-repo `CLAUDE.md` files, with no single entry point. Onboarding therefore depends on the owner's private notes and tacit knowledge — violating the goal of onboarding without meetings. (The owner keeps private working notes elsewhere to brief AI tools; those are explicitly *not* a platform surface and must not be a dependency for contributors.)

## Decision
Create `enpicie-platform` as **the** source of truth for the platform in GitHub — both *what exists / how to use it* (catalog, onboarding, ADRs) and *how projects are built* (the shared standards and the keystone config convention). It is self-contained: a contributor with only a GitHub account can onboard from it alone. `Infrastructure-Documentation` folds into it over time.

## Rationale
- A newcomer with only a GitHub account can understand and use the platform — no private toolchain required.
- One place to update when a piece is added, versioned, or a convention changes.
- Removes the dependency on any private/owner-only surface for contributors.
- Cheap to maintain: catalog tables + runbook + ADRs, all close to the code they describe.

## Consequences
- Risk of catalog staleness. Mitigation: the catalog is a small table; a future CI check can diff it against the org repo list.
- The shared standards must be authored here rather than assumed from private notes (R2 work).

## Supersedes / Superseded By
—
