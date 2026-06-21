# Env & Secrets — Single Source of Truth

> The one convention that persists in **every** enpicie project, regardless of stack or deploy target. If a project deploys to AWS, its secrets live in AWS Secrets Manager. Everything else about a project can vary; this does not.

## The problem this solves

Today, a project's configuration is scattered: some values in `deployment.env`, some in GitHub secrets, some declared ad-hoc in Terraform (`aws_secretsmanager_secret "startgg_api_token"`), and some imported as data sources. Two different patterns for the same thing (`adomi-san-bot` both *creates secret values in Terraform* and *imports pre-populated ones*). That means:
- No single place answers "what config does this app need?"
- Secret **values** can leak into Terraform state (the `secret_version` + `var.*` path).
- Local and prod config drift silently.

## The model

**One manifest per project is the source of truth.** It declares every variable and secret *once* — names, shape, and sensitivity, never values. Everything else is generated or wired from it.

```
                       ┌─────────────────────────┐
                       │   config/app-config.yaml │   ← committed, the SSOT
                       │   (names + shape, no values)
                       └────────────┬─────────────┘
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
      .env.example            Terraform               CI validation
   (local dev template)   (Secrets Manager           (fail fast if a
                           container + IAM +           required key is
                           runtime wiring)             missing)
```

**Values live in exactly two places, never in code or state:**
| Environment | Non-secret vars | Secret vars |
|---|---|---|
| local | `.env.local` (gitignored) | `.env.local` (gitignored) |
| AWS (prod) | plain env (from `deployment.env` / manifest) | **AWS Secrets Manager** |

## The manifest — `config/app-config.yaml`

Committed (it's names and shape, not values). See [templates/app-config.example.yaml](../templates/app-config.example.yaml).

```yaml
app: adomi-san-bot
vars:
  - name: AWS_REGION
    desc: AWS region to deploy into
    secret: false
    scope: [build, runtime]   # where the value is consumed
    required: true
  - name: STARTGG_API_TOKEN
    desc: Start.gg API token
    secret: true              # → AWS Secrets Manager on AWS deploys
    scope: [runtime]
    required: true
  - name: DISCORD_BOT_TOKEN
    desc: Discord bot token
    secret: true
    scope: [runtime]
    required: true
```

## How values resolve, per surface

### AWS Secrets Manager (the storage decision)
- **One JSON secret per app per env**, named `{app}/{env}` (e.g. `adomi-san-bot/prod`), holding a JSON object of all `secret: true` keys.
  - One IAM resource to grant, one fetch at runtime, one place to look. Cheaper than one-secret-per-key (`$0.40/secret/mo`).
  - `adomi-san-bot` currently uses one-secret-per-key — supported, but new projects use the JSON-blob default; adomi can migrate later.
- **Terraform manages the secret *container* + IAM only — never the value.** No `aws_secretsmanager_secret_version` with `var.*` in the committed config. Values are populated out-of-band (see below). This removes the state-leak.

### Runtime injection
- **Lambda:** the [AWS Parameters and Secrets Lambda Extension](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_lambda.html) fetches `{app}/{env}` at runtime and caches it. Secrets never become plaintext Lambda env vars or appear in logs. Non-secret vars are plain Lambda env. (This matches adomi's existing "fetched by the Lambdas at runtime" comment — now standardized.)
- **ECS:** task definition `secrets` block maps each key natively: `valueFrom = "{secret_arn}:KEY::"`. ECS injects them as env at container start. Already compatible with `tf-module-ecs-alb-service`.

### Populating values (out-of-band, never in state)
```bash
# from a filled .env.local, push secret keys to Secrets Manager for an env
make secrets-push ENV=prod        # wraps: aws secretsmanager put-secret-value ...
```
Or set them once in the console. Terraform never sees the value.

### Local development
```bash
make config       # regenerate .env.example from the manifest
cp .env.example .env.local && $EDITOR .env.local   # fill real dev values
make config-check # fail if .env.local is missing any required key
```

## Platform pieces this needs (see ROADMAP Phase 1)
- **`tf-module-app-config`** — input: app + env + the manifest's secret keys. Output: the `{app}/{env}` Secrets Manager container + a scoped `GetSecretValue` IAM policy for the app's execution role. Manages container + IAM only.
- **`gh-action-validate-config`** — reads the manifest, checks every `required` secret key exists in Secrets Manager before deploy; fails fast otherwise.
- **Make targets** — `config`, `config-check`, `secrets-push` (added to the standard Makefile from the General Platform Spec).

## Rules
- The manifest is the only place config is *declared*. If a var isn't in the manifest, it doesn't exist.
- Secret values never appear in: code, committed files, Terraform state, GitHub Actions logs, or Lambda plaintext env.
- On AWS, every `secret: true` var resolves from Secrets Manager. No exceptions.
- Local and prod share the same keys (from the manifest) — only values differ.

See [ADR-003](./decisions/ADR-003-env-secrets-single-source-of-truth.md) for the decision record.
