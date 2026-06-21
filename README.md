# enpicie-platform

**The single source of truth for the enpicie platform.** You are your own platform team — this repo is the front door and the documentation home for the reusable pieces that let you start, deploy, and hand off projects with low cognitive load.

> Status: **Bootstrapping.** Start with the **[Platform Spec](./spec/platform.md)** (the glue). See [ROADMAP.md](./ROADMAP.md) for what's next and [docs/audit-2026-06.md](./docs/audit-2026-06.md) for the founding audit.

## What the platform is for (the product)

Three things, in priority order:

1. **Centralized docs** — everything needed to understand and deploy a project lives here, in GitHub. No external/private surface required to onboard.
2. **A plug-and-play starting pipeline** — copy a starter, change a few values, ship. Flexible per project, not rigid.
3. **One config convention that never changes: a single source of truth for env vars + secrets**, backed by **AWS Secrets Manager** when a project deploys to AWS. → **[docs/env-and-secrets.md](./docs/env-and-secrets.md)** (the keystone).

Everything else about a project can vary by archetype. Those three persist.

---

## The pieces

A set of composable, versioned building blocks so a new project is **configure, not build**:

- **`gh-action-*`** — reusable GitHub Actions (composite workflows) — the CI building blocks
- **`tf-module-*`** — reusable Terraform modules (run via OpenTofu) — the infra building blocks
- **`aws-*`** — core, shared, account-level AWS infrastructure (singletons — see note below)

Apps deploy to AWS via OpenTofu, authenticating with GitHub OIDC and assuming scoped IAM roles. State lives in S3, one file per infrastructure component.

> **`aws-*` repos are intentionally unversioned.** At personal-org scale they are singletons that just need to exist (the shared VPC/ALB/ECS, the IAM roles). They are not products consumed by version — only the `gh-action-*` and `tf-module-*` pieces are version-pinned.

---

## Platform catalog

### Reusable Terraform modules (`tf-module-*`)
| Repo | Provisions | Latest |
|---|---|---|
| [tf-module-s3-cloudfront-website](https://github.com/enpicie/tf-module-s3-cloudfront-website) | Static site: S3 + CloudFront | v1.2.0 |
| [tf-module-ecs-alb-service](https://github.com/enpicie/tf-module-ecs-alb-service) | Container service on shared ECS + ALB | v1.2.0 |
| [tf-module-eventbridge-scheduled-lambda](https://github.com/enpicie/tf-module-eventbridge-scheduled-lambda) | Scheduled Lambda via EventBridge | tag TBD |
| [tf-module-app-config](https://github.com/enpicie/tf-module-app-config) | `{app}/{env}` Secrets Manager container + scoped IAM (the keystone) | v0.1.0 |

### Reusable GitHub Actions (`gh-action-*`)
| Repo | Does | Latest |
|---|---|---|
| [gh-action-workflow-terraform-run](https://github.com/enpicie/gh-action-workflow-terraform-run) | OpenTofu plan/apply via OIDC (the core deploy step) | v1.0.0 |
| [gh-action-workflow-s3-upload](https://github.com/enpicie/gh-action-workflow-s3-upload) | Upload build artifact to S3 (hashed-filename cleanup) | v1.0.1 |
| [gh-action-workflow-build-python-lambda-layer-zip](https://github.com/enpicie/gh-action-workflow-build-python-lambda-layer-zip) | Build + zip Python Lambda layer → S3 | tag TBD |
| [gh-action-workflow-upload-lambda-zip](https://github.com/enpicie/gh-action-workflow-upload-lambda-zip) | Zip Lambda source → S3 | tag TBD |
| [gh-action-workflow-terraform-destroy](https://github.com/enpicie/gh-action-workflow-terraform-destroy) | Gated teardown of applied configs | tag TBD |
| [gh-action-validate-config](https://github.com/enpicie/gh-action-validate-config) | Fail-fast if a required secret is missing pre-deploy | v0.1.0 |

### Core infrastructure (`aws-*`, singletons, unversioned by design)
| Repo | Purpose |
|---|---|
| [aws-infra](https://github.com/enpicie/aws-infra) | Shared VPC, ALB, ECS cluster |
| [aws-tf-iam-roles](https://github.com/enpicie/aws-tf-iam-roles) | Scoped IAM roles for Terraform runs (ARNs → org secrets) |

### Apps & archetypes
| Repo | Stack | Status / archetype |
|---|---|---|
| [find-my-fgc](https://github.com/enpicie/find-my-fgc) | TS frontend + backend on AWS | Shipped — **reference impl** (`static-frontend` + AWS backend) |
| [adomi-san-bot](https://github.com/enpicie/adomi-san-bot) | Python Discord bot, Lambda + scheduled job | Shipped — `bot` / `scheduled-job` |
| [fgc-league-sheets](https://github.com/enpicie/fgc-league-sheets) | TS Google Apps Script (clasp): React sidebar + server | Shipped — `apps-script` (clasp push, no AWS/TF) |
| [seeding-mcp-server](https://github.com/enpicie/seeding-mcp-server) | Python MCP server | Shipped |
| seeding tool (new) | Full-stack TS (frontend + API) | Planned — Vercel/Railway candidate ([ADR-002](./docs/decisions/ADR-002-deployment-target-decision-tree.md)) |
| [dia](https://github.com/enpicie/dia) | Turborepo TS | **Experimental — actively rebuilding, ignore for now** |

---

## Onboarding: deploy a new project

1. **Pick an archetype** — `static-frontend`, `container-backend`, `scheduled-job`, `full-stack-js`, `bot`, `apps-script`.
2. **Confirm AWS account bootstrap exists** (AWS projects) — S3 state bucket + DynamoDB lock + GitHub OIDC provider + scoped IAM role ARNs in org secrets. See [docs/bootstrap.md](./docs/bootstrap.md) *(planned)*.
3. **Scaffold from the archetype starter** in [templates/](./templates) *(growing)*.
4. **Declare config once** in `config/app-config.yaml` → [the keystone](./docs/env-and-secrets.md). Set `.github/deployment.env` (`APP_NAME`, `AWS_REGION`, `DEPLOYMENT_ENV`).
5. **Push secret values** to AWS Secrets Manager (`make secrets-push`) — never into code or state.
6. **Tag a release** — the pipeline runs `config → build → deploy`.

For full-stack JS apps, the target may instead be **Vercel + Railway + Supabase** — see [ADR-002](./docs/decisions/ADR-002-deployment-target-decision-tree.md).

---

## Docs map
- **[spec/platform.md](./spec/platform.md)** — the platform spec: the glue an agent (or human) follows to build/deploy/onboard a project
- [docs/env-and-secrets.md](./docs/env-and-secrets.md) — the keystone config convention
- [docs/decisions/](./docs/decisions) — ADRs (cross-repo architecture decisions; see [ADR-004](./docs/decisions/ADR-004-spec-driven-glue.md) for the spec-vs-code thesis)
- [ROADMAP.md](./ROADMAP.md) · [docs/audit-2026-06.md](./docs/audit-2026-06.md) (founding audit snapshot)
- [docs/infrastructure-reference.md](./docs/infrastructure-reference.md) — deep AWS infra reference (architecture diagram, OIDC, OpenTofu migration); folded in from the former `Infrastructure-Documentation` repo
