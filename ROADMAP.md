# Platform Roadmap

Sequenced backlog to reach the desired state: **centralized docs + a plug-and-play starter pipeline + one config convention that never changes.** Each item maps to a recommendation in the [founding audit](./docs/audit-2026-06.md). Ordered so early work de-risks later work.

> **Glue decision ([ADR-004](./docs/decisions/ADR-004-spec-driven-glue.md)):** the plug-and-play layer is a **spec an agent executes** ([spec/platform.md](./spec/platform.md)), not a rigid template — over a thin floor of versioned coded primitives. "Phase 3 starter" below means starter *configs the spec generates*, not frozen copies.

---

## Phase 1 — The config keystone (now, highest leverage)
**Goal: one source of truth for env + secrets in every project. (R1 → G2)**

Design is written: [docs/env-and-secrets.md](./docs/env-and-secrets.md) · [ADR-003](./docs/decisions/ADR-003-env-secrets-single-source-of-truth.md) · manifest template in [templates/app-config.example.yaml](./templates/app-config.example.yaml).

- [x] Build **[`tf-module-app-config`](https://github.com/enpicie/tf-module-app-config)** `@v0.1.0` — `{app}/{env}` Secrets Manager container + scoped `GetSecretValue` IAM. Container + IAM only; never the value.
- [x] Build **[`gh-action-validate-config`](https://github.com/enpicie/gh-action-validate-config)** `@v0.1.0` — fails the deploy if a `required` secret key is missing from Secrets Manager.
- [ ] Add Make targets to the standard Makefile: `config` (regen `.env.example` from manifest), `config-check` (`.env.local` has all required keys), `secrets-push` (push secret values from `.env.local` to Secrets Manager).
- [ ] Migrate **`adomi-san-bot`** off the value-in-state pattern → manifest + module (draft PR open; needs the owner's `terraform plan`/apply + runtime-fetch code change to land — see report).

Done when: adding a secret = one manifest line + `make secrets-push`, and no secret value touches code or Terraform state in any project.

---

## Phase 2 — Centralized docs home (in progress)
**Goal: onboard from GitHub alone, no private notes. (R2 → G3)**

- [x] Stand up the repo with catalog, onboarding, ADRs, keystone doc
- [x] Draft the **Platform Spec** (`spec/platform.md`) — the glue
- [ ] **Rename `Infrastructure-Documentation` → `platform`** (unused v1; near-zero cost) and push this content into it
- [ ] Fold the existing Infra-Doc README (architecture diagram, OIDC, OpenTofu migration) into `docs/` + `docs/diagrams/`
- [ ] Author the remaining shared standards (coding / API / design) under `spec/` so they're self-contained, not assumed from private notes
- [ ] Pin the repo in the org; link to it from every app README
- [ ] Fill the `tag TBD` cells in the README catalog
- [ ] Link to this repo from every app README

Done when: a new person lands on the org and can understand + deploy from this repo alone.

---

## Phase 3 — Plug-and-play starter pipeline
**Goal: copy + adjust, ship. (R3 → G5)**

- [ ] Build a starter under `templates/<archetype>/` per archetype, each pre-wired with: `config/app-config.yaml`, the 3-file workflow (`release` + `workflow-build` + `workflow-deploy`) + `deployment.env`, a `terraform/` skeleton, `Makefile` (incl. config targets), and `CLAUDE.md`.
- [ ] Make it flexible by design — a copyable starter you adjust, not a rigid generator. Optionally a thin `create-enpicie-app` script that copies a starter and prompts for `APP_NAME`/region.
- [ ] Verify: a person with only git + the language runtime ships a hello-world to prod following only `docs/`.

Done when: starting a project = "copy the starter, fill three values + the manifest, push secrets, tag a release."

---

## Phase 4 — One pattern per archetype
**Goal: finite, named CI shapes — no snowflakes. (R4 → G1)**

- [ ] Name the archetypes: `static-frontend`, `container-backend`, `scheduled-job`, `full-stack-js`, `bot`, `apps-script`
- [ ] One reference repo per archetype (find-my-fgc = `static-frontend`+AWS; fgc-league-sheets = `apps-script`; adomi-san-bot = `bot`/`scheduled-job`)
- [ ] For each shipped app, conform to its named pattern or document the blessed variant
- [ ] `docs/archetypes/<name>.md` per archetype

Done when: every app maps to exactly one named archetype, all documented here.

---

## Phase 5 — Where new apps live
**Goal: stop defaulting JS apps onto heavy AWS. (R5 → G6)**

- [ ] Finalize [ADR-002](./docs/decisions/ADR-002-deployment-target-decision-tree.md) (currently Proposed)
- [ ] Pilot on the **seeding tool**: frontend on **Vercel**, API on **Railway** (or Vercel functions), db on **Supabase** — config still via the keystone manifest, resolving from the platform's native secret store
- [ ] Add `docs/stacks/vercel.md` + `docs/stacks/railway.md`
- [ ] Revisit `dia` against ADR-002 once its rebuild stabilizes (experimental — out of scope now)

Done when: `git push` deploys the seeding tool with zero pipeline babysitting, and the target-selection rule is written down.

---

## Phase 6 — Cleanups (anytime)
**(R6 → G4)**

- [ ] Standardize default `AWS_REGION`; note exceptions in `deployment.env`
- [ ] Fix the CI/CD doc link: `action-workflow-build-...` → `gh-action-workflow-build-python-lambda-layer-zip`
- [ ] Close the Infrastructure-Documentation TODO: manage `adomi-san-bot` Lambda zip via Terraform
- [ ] Backfill missing version tags on `gh-action-*` repos referenced by version

---

## Not doing (yet) — with reason
- **Versioning the `aws-*` repos** — intentional singletons at personal-org scale; they just need to exist, not be consumed by version.
- **Staging environments** — local+prod only at MVP; revisit per app in `scaling-plan.md`.
- **Multi-cloud abstraction** — premature; the AWS vs Vercel/Railway split (ADR-002) is the only portability that pays off now.
- **Custom monitoring/usage tables** — use platform-native observability, never DB tables.
