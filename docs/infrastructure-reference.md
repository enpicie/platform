# Infrastructure Reference

> Deep reference for the `enpicie` AWS infrastructure. Folded in from the former `Infrastructure-Documentation` repo. For how to *build* a project, start with the [Platform Spec](../spec/platform.md); for config, see [env-and-secrets](./env-and-secrets.md).

## Core infrastructure stack
- **CI/CD pipelines:** GitHub Actions
- **Cloud provider:** AWS
- **Provisioning:** Terraform via OpenTofu

## Goals
- Automate deployments and consume reusable pipeline stages for repeated work
- Minimize manual steps to obtain authorization for each new project
- Minimize manual steps to update permissions or alter infrastructure config
- Avoid committing or exposing sensitive data / credentials

## Architecture diagram

![Architecture Diagram](./diagrams/Deployment-Infrastructure.drawio.png)

- **[gh-action-workflow-terraform-run](https://github.com/enpicie/gh-action-workflow-terraform-run)** — the core reusable GH Actions workflow for Terraform runs across all projects
  - Owns CloudFormation templates that provision the prerequisite backend for Terraform state management
  - Lets projects pass Terraform variables via `TF_VAR_*` env vars as needed
- **[aws-tf-iam-roles](https://github.com/enpicie/aws-tf-iam-roles)** — Terraform config for the IAM roles assumed during Terraform runs, scoped by use case
  - Creates roles with permissions for different sets of AWS services
  - Role ARNs are saved as GitHub **org** secrets for projects to consume in Actions
  - New role/permission needs are added here
- **[aws-infra](https://github.com/enpicie/aws-infra)** — Terraform config for core shared infrastructure
  - Primarily VPC, ALB, and ECS cluster
  - Only houses shared, app-independent resources

All core repos above use one-time, manually created IAM roles for the permissions needed to provision their own resources. (These `aws-*` repos are intentionally unversioned singletons — see [README](../README.md).)

## GitHub Actions
Workflows compose into pipelines that encapsulate reusable units of work across projects. The core deploy workflow is [gh-action-workflow-terraform-run](https://github.com/enpicie/gh-action-workflow-terraform-run). Repos named `gh-action-workflow-*` are composite workflows; each documents itself in its own README.

- [gh-action-workflow-terraform-run](https://github.com/enpicie/gh-action-workflow-terraform-run) — run Terraform for a given config
- [gh-action-workflow-build-python-lambda-layer-zip](https://github.com/enpicie/gh-action-workflow-build-python-lambda-layer-zip) — build the `.zip` for a Python Lambda layer (runtime deps)
- [gh-action-workflow-upload-lambda-zip](https://github.com/enpicie/gh-action-workflow-upload-lambda-zip) — upload a Lambda `.zip` to S3
  - **Prefer Terraform where possible.** _TODO: determine if `adomi-san-bot` can manage the Lambda `.zip` via Terraform alone._

## OIDC setup
All roles share a trust relationship letting GitHub Actions authenticate to AWS and assume a role by ARN. Since HCP Terraform is no longer used, AWS only needs trust with the GitHub Actions agent via the GitHub OIDC provider. _This OIDC provider was created manually in AWS._

## Migration to [OpenTofu](https://opentofu.org/)
OpenTofu is built on Terraform to keep Terraform usable as HashiCorp's licensing changes — effectively a wrapper for the Terraform CLI. The migration from HCP Terraform to OpenTofu via GitHub Actions removed the HCP layer and made runs more direct.
