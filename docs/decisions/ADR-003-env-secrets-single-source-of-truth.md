# ADR-003: Env & secrets single source of truth (AWS Secrets Manager on AWS)

## Status
Accepted

## Date
2026-06-21

## Context
Configuration is the one thing that must behave identically across every project for the platform to feel coherent. Today it doesn't. `adomi-san-bot` shows the current reality: secrets are declared ad-hoc per resource, with two contradictory patterns in the same file — some secret **values** are written through Terraform (`aws_secretsmanager_secret_version` fed by `var.startgg_api_key`), which lands the value in Terraform state, while others are imported as pre-populated data sources. There is no single declaration of "what config does this app need," and local vs prod config can drift.

The platform owner has named this the keystone: *the core thing that must persist in all projects is a single source of truth for env vars and secrets, managed via AWS Secrets Manager when deployed to AWS.*

## Decision
Adopt a **manifest-driven** config model. Every project commits `config/app-config.yaml` declaring every variable and secret once — names, shape, sensitivity — but **never values**. The manifest drives `.env.example` generation, Terraform secret provisioning, runtime wiring, and CI validation.

Concrete decisions:
1. **Storage:** secrets live in AWS Secrets Manager as **one JSON secret per app per env**, named `{app}/{env}`. (Alternative considered: one secret per key, as adomi does today — rejected as default for higher cost and more IAM/fetch surface; still supported, adomi may migrate.)
2. **Terraform manages the secret container + IAM only, never the value.** No committed `secret_version` sourced from a `var.*`. This removes the state-leak.
3. **Values are populated out-of-band** — `make secrets-push` (wrapping `aws secretsmanager put-secret-value`) or the console. Terraform never sees them.
4. **Runtime injection without plaintext env:** Lambda uses the AWS Parameters and Secrets Lambda Extension to fetch at runtime; ECS uses native `secrets`/`valueFrom`. Non-secret vars may be plain env.
5. **Local = prod keys, different values:** `.env.local` (gitignored) holds local values; `make config-check` fails if it's missing a required key.
6. **New platform pieces:** `tf-module-app-config` (container + scoped IAM) and `gh-action-validate-config` (fail-fast on missing required secrets), plus `config` / `config-check` / `secrets-push` Make targets.

This generalizes what adomi already does right (runtime fetch, single-sourced names, scoped IAM) and removes what it does inconsistently.

## Rationale
- **Single source of truth:** one manifest answers "what does this app need," and everything else is derived — no scattered declarations.
- **No secret leakage:** values never touch code, state, logs, or plaintext env.
- **Plug-and-play across stacks:** the manifest is stack-agnostic; only the resolver (Lambda extension vs ECS valueFrom vs local `.env`) differs, and those are platform-provided.
- **Low maintenance:** adding a secret = one manifest line + one `secrets-push`. No bespoke Terraform per secret.

## Consequences
- A new module + action to build and maintain (Phase 1 of the roadmap).
- Projects not on AWS (e.g. `fgc-league-sheets` on Apps Script, or future Vercel/Railway apps) still use the same manifest for *declaration*, but resolve secrets via that platform's native secret store. The manifest stays the SSOT; only the resolver changes. Document the non-AWS resolver in each stack's doc.
- `adomi-san-bot` is non-conformant until migrated (per-key secrets, value-in-state). Tracked as a remediation, not a blocker.

## Supersedes / Superseded By
—
