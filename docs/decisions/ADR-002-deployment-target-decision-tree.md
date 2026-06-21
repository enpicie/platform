# ADR-002: Deployment-target decision tree (AWS vs Vercel vs Railway)

## Status
Proposed

## Date
2026-06-21

## Context
The platform is AWS-by-default: S3 + CloudFront for frontends, Lambda/API Gateway or ECS/ALB for backends, all via OpenTofu + Terraform. This is the right call for infrastructure you need to *own* (fine-grained IAM, scheduled jobs, FGC tooling tied to AWS resources).

The next planned full-stack TypeScript app is the **seeding tool** (the `dia` rebuild is experimental and out of scope until it stabilizes). For a push-to-deploy web app, the AWS path carries more setup and maintenance than the app needs, which works against the goal of low-maintenance, fast-to-start projects.

There is currently no written rule for *which* target a new project uses, so every new app re-litigates the question or silently diverges. The platform also already ships an app on a target the spec doesn't mention at all — `fgc-league-sheets`, a Google Apps Script extension deployed via `clasp push`.

## Decision
Adopt an explicit decision tree. Default new **full-stack JS/TS apps** to managed platforms; reserve **AWS + Terraform** for infrastructure that needs ownership.

```
New project
│
├─ Needs fine-grained AWS IAM, VPC, scheduled Lambdas,
│  or integrates with existing AWS resources (FGC tooling)?
│        └─ YES → AWS + OpenTofu (existing platform path)
│
├─ Full-stack web/JS app, wants push-to-deploy, low ops?
│        ├─ Frontend / Next-style / SSR        → Vercel
│        ├─ Long-running API / worker / queue   → Railway
│        └─ Postgres / auth / storage           → Supabase
│
├─ Python service / MCP server / bot?
│        ├─ Event-driven, scheduled, or AWS-tied → AWS (Lambda + tf-module-eventbridge-scheduled-lambda)
│        └─ Always-on, simple                     → Railway
│
└─ Google Sheets / Workspace extension?
         └─ Apps Script via clasp (own archetype; no AWS/TF) — e.g. fgc-league-sheets
```

Apply to the apps:
- **seeding tool** (next up) → Vercel (web) + Railway (API) + Supabase (db); `seeding-mcp-server` stays as-is or moves to Railway if always-on.
- **dia** → revisit against this tree once the rebuild stabilizes (experimental, out of scope now).
- **find-my-fgc, adomi-san-bot, FGC tooling** → stay on AWS.
- **fgc-league-sheets** → stays on Apps Script/clasp (its own archetype).

Add thin **Vercel** and **Railway** stack docs under this repo's `docs/` so projects (and AI tooling) initialize them correctly — env handling via the keystone manifest (resolving from each platform's native secret store instead of AWS Secrets Manager), preview deploys, and structured logging via platform-native tooling.

## Rationale
- **Maintenance:** Vercel/Railway give push-to-deploy, preview environments, and built-in observability — directly serving "don't babysit pipelines." The AWS path's value (IAM scoping, state management) is overhead these apps don't need.
- **Onboarding:** "connect repo, set env vars" is a smaller surface than the OIDC + scoped-role + per-component-state model.
- **Cost at this scale:** both have usable free/hobby tiers; cost crosses over only at scale, which `scaling-plan.md` already governs.
- **Keeps strengths:** the mature AWS platform stays the home for the things it's genuinely best at — no rip-and-replace.

## Consequences
- A second deployment surface to understand. Mitigated by writing it down here and in stack specs, and by routing each project deterministically via the tree.
- Two observability/secrets stories (AWS-native vs Vercel/Railway-native). Document both in the relevant stack specs.
- The General Platform Spec's CI/CD section currently assumes AWS; it must note that managed-platform projects follow their stack spec instead of `spec-platform-cicd.md`.

## Open questions (decide during the seeding-tool pilot)
- Vercel functions vs Railway for the API — depends on whether the API is request/response or long-running.
- Whether Supabase replaces or complements current data stores per app.

## Supersedes / Superseded By
Refines, does not supersede, the CI/CD spec — adds a target-selection step before it.
