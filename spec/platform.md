# enpicie Platform Spec

*The entry spec for building, deploying, and onboarding any project on the enpicie platform.*
*Read this first. It composes the standards and routes you to the right reusable pieces.*
*Audience: an AI coding agent (primary) and a human contributor (same instructions, read as a runbook).*

---

## 1. Purpose & how to use this spec

This spec is the **glue** of the platform. It does not contain infrastructure logic — it tells you which versioned pieces to use and how to assemble them into a conformant project. Follow it top to bottom when initializing a project; apply §10 (Audit Mode) when conforming an existing one.

The platform's design principle ([ADR-004](../docs/decisions/ADR-004-spec-driven-glue.md)):

> **Reference the primitives, don't regenerate them. Spec for the assembly; code for the primitives.**

The reusable `gh-action-*` and `tf-module-*` repos are version-pinned, security-reviewed building blocks. **Never reimplement their logic in a project.** Reference them by tag. If a primitive is missing, that is a platform gap — raise it; do not hand-roll a one-off.

---

## 2. The primitive floor — reference, never rebuild

Pin every reference to a version tag. Never `@main`.

### GitHub Actions (`gh-action-*`)
| Action | Use for | Pin |
|---|---|---|
| `enpicie/gh-action-workflow-terraform-run` | OpenTofu plan/apply via OIDC — **all** infra changes | `@v1.0.0` |
| `enpicie/gh-action-workflow-s3-upload` | Frontend dist → S3 (hashed-filename cleanup) | `@v1.0.1` |
| `enpicie/gh-action-workflow-build-python-lambda-layer-zip` | Build/zip Python Lambda layer → S3 | pin latest tag |
| `enpicie/gh-action-workflow-upload-lambda-zip` | Zip Lambda source → S3 | pin latest tag |
| `enpicie/gh-action-workflow-terraform-destroy` | Gated teardown | pin latest tag |
| `enpicie/gh-action-validate-config` *(planned)* | Fail-fast if a required secret is missing pre-deploy | pin when built |

### Terraform modules (`tf-module-*`)
| Module | Provisions | Pin |
|---|---|---|
| `tf-module-s3-cloudfront-website` | Static site: S3 + CloudFront | `@v1.2.0` |
| `tf-module-ecs-alb-service` | Container service on shared ECS + ALB | `@v1.2.0` |
| `tf-module-eventbridge-scheduled-lambda` | Scheduled Lambda via EventBridge | pin latest tag |
| `tf-module-app-config` *(planned)* | `{app}/{env}` Secrets Manager container + scoped IAM (keystone) | pin when built |

### Core infrastructure (`aws-*`, singletons — assume they exist, never recreate)
- `aws-infra` — shared VPC, ALB, ECS cluster.
- `aws-tf-iam-roles` — scoped IAM roles for Terraform runs; ARNs are GitHub **org** secrets.

If a project needs a new IAM permission set, add a role in `aws-tf-iam-roles` — never widen an existing role or create an inline one in the project.

---

## 3. Step 1 — Choose the deployment target

Per [ADR-002](../docs/decisions/ADR-002-deployment-target-decision-tree.md):

```
Needs fine-grained AWS IAM / VPC / scheduled Lambdas, or integrates with existing AWS? → AWS + OpenTofu
Full-stack web/JS app wanting push-to-deploy, low ops?                                 → Vercel (web) + Railway (API) + Supabase (db)
Python service / MCP / bot? event-driven or AWS-tied → AWS; always-on & simple         → Railway
Google Sheets / Workspace extension?                                                   → Apps Script via clasp
```

The rest of this spec details the **AWS path** (the platform's core). Managed-platform projects (Vercel/Railway) follow their stack doc under `docs/stacks/` but **still use the §5 config keystone** — only the secret resolver changes (the platform's native secret store instead of AWS Secrets Manager).

---

## 4. Step 2 — Choose the archetype

Every project maps to exactly one. The archetype determines which primitives you reference.

| Archetype | Reference repo | Primitives |
|---|---|---|
| `static-frontend` | find-my-fgc | `tf-module-s3-cloudfront-website`, `gh-action-workflow-s3-upload` |
| `container-backend` | — | `tf-module-ecs-alb-service` |
| `scheduled-job` | adomi-san-bot | `tf-module-eventbridge-scheduled-lambda`, lambda build/upload actions |
| `bot` | adomi-san-bot | lambda build/upload actions, `tf-module-eventbridge-scheduled-lambda` |
| `full-stack-js` | seeding tool (planned) | Vercel + Railway + Supabase (ADR-002) |
| `apps-script` | fgc-league-sheets | clasp (`deploy-to-sheet.yml`); no AWS/Terraform |

If a project doesn't fit a named archetype, stop and define a new one (with a reference repo) rather than improvising a snowflake.

---

## 5. Step 3 — Apply the config keystone (non-negotiable, every project)

This is the one convention that never changes. Full detail: [docs/env-and-secrets.md](../docs/env-and-secrets.md). Binding rules:

- Generate `config/app-config.yaml` — the single source of truth declaring every var + secret (names + shape, **never values**).
- On AWS, every `secret: true` var resolves from **AWS Secrets Manager**, stored as one JSON secret named `{app}/{env}`.
- **Terraform manages the secret container + IAM only — never the value.** No `secret_version` fed by a `var.*`. Values are pushed out-of-band (`make secrets-push`).
- Secrets are **fetched at runtime** (Lambda Parameters/Secrets extension; ECS `valueFrom`) — never plaintext env, never in logs, never in state.
- Local dev uses `.env.local` (gitignored) with the same keys; `make config-check` fails if a required key is missing.

Reference `tf-module-app-config` for the container + IAM. Do not write bespoke `aws_secretsmanager_secret` resources per secret.

---

## 6. Step 4 — Scaffold

Generate the platform baseline (this composes the General Platform Spec's init requirements — `CLAUDE.md`, `docs/`, `README.md`, `Makefile`, `.env.example`, ADRs, `.gitignore`). Platform-specific files:

```
.github/
  deployment.env              # APP_NAME, AWS_REGION, DEPLOYMENT_ENV (non-secret only)
  workflows/
    release.yml               # config -> build -> deploy (tags + workflow_dispatch)
    workflow-build.yml        # reusable build (workflow_call)
    workflow-deploy.yml       # reusable deploy — all Terraform runs (workflow_call)
config/
  app-config.yaml             # the keystone manifest (§5)
terraform/
  <component>/                # one root per component: frontend/, backend/, etc.
    main.tf                   # references tf-module-* by version; no inlined infra
    variables.tf             # every TF_VAR declared; no defaults for required
    outputs.tf               # values consumed by later workflow steps
Makefile                      # dev, test, lint, build + config, config-check, secrets-push
```

Terraform rules: empty S3 backend block (`backend "s3" {}`), state key `projects/{app}/{env}/{component}.tfstate`, one IAM role per component (from `aws-tf-iam-roles`), read resource identifiers from module outputs — never hardcode bucket names, distribution IDs, or account IDs.

---

## 7. Step 5 — Assemble CI/CD

Wire the three-file workflow, referencing actions by version. Key invariants:
- `deployment.env` is the only place project config values live; the `config` job sources it and passes them downstream.
- `app_version` is always `github.sha` (not the tag).
- IAM role ARNs are **secrets**, never `inputs` (inputs are logged, secrets are masked).
- A deploy commonly runs Terraform once per component, each with its own scoped role + state key.
- Plan only on PRs; apply only on merge to main: `apply: ${{ github.ref == 'refs/heads/main' && 'true' || 'false' }}`.
- Job-level permissions: `id-token: write`, `contents: read`.

The detailed workflow shapes per archetype live in `docs/archetypes/<name>.md`.

---

## 8. Security rules — non-negotiable

- Never hand-roll IAM trust/permission policies, OIDC, or the Terraform state backend — reference the primitives.
- Never put a secret value in code, a committed file, Terraform state, Actions logs, or plaintext env.
- Never use a single IAM role for a whole project — scope per component.
- Never pin a primitive to `@main` — always a version tag.
- Never apply Terraform from a PR branch.

---

## 9. Anti-patterns — never do these

- **Reimplementing a primitive's logic inside a project.** Reference it. Missing capability → platform gap, not a local copy.
- **Bespoke per-secret Terraform.** Use the keystone manifest + `tf-module-app-config`.
- **Snowflake CI.** Every project maps to a named archetype's pattern.
- **Hardcoding resource identifiers** Terraform manages — read from outputs.
- **Observability via DB tables.** Use platform-native monitoring.
- **"Follow best practices" as an instruction.** Name the piece, the pattern, the reason.

---

## 10. Audit mode — applying this spec to an existing repo

1. Read this spec and the keystone doc in full.
2. Identify the project's archetype and deployment target.
3. Produce `docs/platform-audit.md`: standards met / partially met (specific gaps) / not met (remediation).
4. Flag, in priority order: secret values in state, un-pinned (`@main`) primitive refs, reimplemented primitive logic, missing config manifest, snowflake CI.
5. Ask which gaps to auto-fix vs flag; apply approved fixes in one commit: `chore: platform conformance`.

---

## 11. Onboarding contract

A project is conformant only if a contributor with **only** a GitHub account, git, and the language runtime can: understand the project from its `CLAUDE.md` + `README.md` + this spec; deploy by setting `deployment.env`, filling the manifest, pushing secrets, and tagging a release — with **no meeting and no private knowledge required.**

---

*enpicie Platform Spec — v0.1 (draft). The glue. Composes the General Platform / Coding / API / Design standards; references the versioned primitive floor.*
*Decisions: [ADR-001](../docs/decisions/ADR-001-platform-home-repo.md) · [ADR-002](../docs/decisions/ADR-002-deployment-target-decision-tree.md) · [ADR-003](../docs/decisions/ADR-003-env-secrets-single-source-of-truth.md) · [ADR-004](../docs/decisions/ADR-004-spec-driven-glue.md).*
