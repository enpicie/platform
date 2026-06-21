# ADR-004: Spec-driven platform glue over a coded primitive floor

## Status
Accepted

## Date
2026-06-21

## Context
With the reusable pieces (`gh-action-*`, `tf-module-*`) already built and versioned, the open question was the nature of the *glue* that turns those pieces into a whole project: should it be **coded artifacts** (a template repo or a generator) or a **spec that an AI agent executes** to build a platform-conformant project?

The platform's goals — flexible per project, low maintenance, onboard-without-meetings, AI-assisted by default — push toward spec. But "everything is a spec" has a real failure mode: asking an AI to re-derive IAM trust policies, OIDC wiring, and Secrets Manager access per project invites security drift and loses the propagate-by-version-bump property the coded pieces already give.

## Decision
The platform glue is a **spec** — [`spec/platform.md`](../../spec/platform.md) — that an AI agent reads to scaffold, deploy, and onboard a project. Beneath it sits a **thin, versioned floor of coded primitives** (`gh-action-*`, `tf-module-*`, and the secrets+IAM module) that the spec **references by version and never regenerates.**

The boundary rule:
- **Code it** when it must be identical everywhere, is security-sensitive, and should be version-pinned. The spec says *"reference `@v1.0.0`,"* not *"build this."*
- **Spec it** when it varies per project, benefits from adaptation, and an AI can re-derive it reliably from a contract — file structure, glue, archetype assembly, wiring.

> Spec for the assembly. Code for the primitives. The spec's power comes from having stable, trustworthy things to point at.

## Rationale
- **Flexibility:** the agent adapts each project rather than inheriting a frozen template that goes stale on first edit.
- **Currency:** projects built later automatically reflect the latest spec.
- **Security & propagation:** a fix to a coded primitive propagates by a version bump; an AI re-deriving IAM per project gives no such guarantee. Keeping the security-critical floor coded preserves this.
- **PoC value:** demonstrates a defensible thesis — a spec-driven platform that knows exactly where specs stop and code starts — rather than the naive "AI generates all my infra."

## Consequences
- The spec becomes the primary maintained product; it must stay accurate to the coded floor it references.
- The coded floor stays deliberately minimal and audited.
- Non-AI contributors follow the same spec as a human runbook — it must read as both.

## Alternatives rejected
- **Template repo / generator** — drifts the moment a project is adjusted; copies go stale; security fixes don't propagate.
- **Pure spec, no coded floor** — AI re-derives security-critical IAM/OIDC per project → drift and risk; loses version-bump propagation.

## Supersedes / Superseded By
Establishes the principle that [ADR-001](./ADR-001-platform-home-repo.md) (docs home) and [ADR-003](./ADR-003-env-secrets-single-source-of-truth.md) (config keystone) operate under.
