## Project
enpicie-platform — the platform home / front door for the `enpicie` GitHub org.
This is a docs + meta repo, not an application. It catalogs reusable pieces and encodes platform decisions.

## What this repo is for
- **The** GitHub source of truth for the platform — self-contained, no private notes required to onboard.
- Catalog of `gh-action-*`, `tf-module-*`, `aws-*` repos with current versions.
- The keystone config convention (env + secrets SSOT) — `docs/env-and-secrets.md`.
- Plug-and-play starter pipelines (`templates/`) and per-archetype docs.
- Onboarding runbook + org-level ADRs (cross-repo decisions).

## The product (priority order)
1. Centralized docs (here, GitHub-only).
2. A plug-and-play starter pipeline, adjustable per project.
3. One config convention that never changes: env/secrets SSOT via AWS Secrets Manager on AWS.

## Source of truth
- This repo is authoritative for the platform. Do **not** treat any private/owner notes as a platform surface or a dependency for contributors.
- `aws-*` repos are intentional unversioned singletons (just need to exist).
- Deep infra reference lives at `docs/infrastructure-reference.md` (folded in from the former `Infrastructure-Documentation` repo).

## Conventions
- ADRs live in `docs/decisions/` using the General Platform Spec ADR format. New cross-repo decision → new ADR, never edit an accepted one (mark Superseded).
- The README catalog tables are the canonical list of platform pieces. Update them when a repo is added or tagged.
- Audit follows the Platform Spec Audit Mode (`spec/platform.md` §10). Snapshot in `docs/audit-2026-06.md`; re-run per ROADMAP phase.

## Key files
- `spec/platform.md` — the platform spec (the glue) — start here
- `README.md` — catalog + onboarding entry point
- `docs/env-and-secrets.md` — the env/secrets keystone convention
- `ROADMAP.md` — sequenced backlog
- `docs/decisions/` — ADRs
- `docs/audit-2026-06.md` — founding audit snapshot

## Do not modify without explicit instruction
- Accepted ADRs (supersede instead).

## Initialized with
Platform standards applied in Audit Mode against the live `enpicie` org.
