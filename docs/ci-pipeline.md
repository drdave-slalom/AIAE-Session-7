# CI Pipeline

The Todo Service CI is split into a reusable workflow and a small caller workflow. This lets service teams adopt the same validation and deployment path without copying the job definitions.

## Reusable Workflow

[`.github/workflows/golden-path-ci.yml`](../.github/workflows/golden-path-ci.yml) is triggered with `workflow_call`. It accepts these inputs:

| Input | Default | Purpose |
|---|---:|---|
| `node_version` | `20` | Node.js version for lint and test jobs |
| `terraform_version` | `1.7.0` | Terraform version for infrastructure jobs |
| `run_terraform_plan` | `false` | Enables Checkov and Terraform plan |
| `run_terraform_apply` | `false` | Enables Terraform apply after the plan |
| `build_and_push` | `false` | Enables ECR image publishing and ECS deployment |

The workflow defines these jobs:

- **`lint`** checks the backend and frontend JavaScript with their workspace ESLint scripts. It catches style and static-analysis issues before code is merged.
- **`test`** installs dependencies and runs the backend Jest suite with coverage. Jest enforces the repository's 80% coverage threshold, and the job writes line, branch, function, and statement coverage to the GitHub step summary.
- **`security-scan`** runs when `run_terraform_plan` is enabled. Checkov scans `infra/` and fails on HIGH-severity findings so infrastructure policy violations do not pass CI.
- **`terraform-plan`** runs when `run_terraform_plan` is enabled. It authenticates to AWS with OIDC, installs the requested Terraform version, initializes the dev stack with its CI S3 state key, creates a plan artifact, and adds a plan summary to the GitHub step summary.
- **`docker-build`** runs on pull requests after `lint` and `test`. It builds both backend and frontend Dockerfiles without pushing images, validating that the deployable artifacts can be built.
- **`terraform-apply`** runs when `run_terraform_apply` is enabled and depends on the saved plan. It switches the dev stack to real OIDC credentials, initializes the S3 backend, downloads the plan artifact, applies it, and reports the deployed load balancer URL.
- **`build-and-push`** runs when `build_and_push` is enabled and depends on `terraform-apply`. It resolves the ECR repositories, authenticates to ECR, builds and pushes both images with the commit SHA and `latest` tags, and triggers an ECS service deployment.

The reusable workflow grants read access to repository contents and pull-request write access at workflow scope. AWS jobs request `id-token: write` and use short-lived OIDC credentials instead of long-lived AWS keys.

## Adopting The Workflow

A service team adds a caller workflow at `.github/workflows/todo-service-ci.yml`. The minimum caller for pull-request validation and main-branch deployment is:

```yaml
name: Todo Service CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  call-golden-path:
    uses: ./.github/workflows/golden-path-ci.yml
    with:
      node_version: "20"
      run_terraform_plan: true
      run_terraform_apply: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
      build_and_push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
    secrets:
      aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

`id-token: write` must be present in the caller's top-level permissions. GitHub determines whether the caller may request an OIDC token before invoking the reusable workflow.

## Required Checks

- **Lint** is required to keep backend and frontend code consistent and catch static issues early.
- **Test** is required to verify backend behavior and enforce the 80% Jest coverage threshold.
- **Security scan** is required when infrastructure validation is enabled to detect HIGH-severity Terraform policy violations.
- **Terraform plan** is required when infrastructure validation is enabled to verify that the dev stack initializes and produces an infrastructure change plan before apply.
- **Docker build** runs on pull requests to catch broken backend or frontend container builds before merge.

The caller enables `run_terraform_plan: true`, so security scanning and Terraform planning run for both pull requests and pushes to `main`. Apply and image publishing are limited to pushes to `main` by the caller expressions.

## OIDC Secret Configuration

The Terraform plan, apply, and image publishing jobs receive the AWS role through the reusable workflow secret named `aws_role_arn`. Configure it as a repository Actions secret named `AWS_ROLE_ARN`:

1. Open the repository on GitHub.
2. Go to **Settings > Secrets and variables > Actions**.
3. Create a new repository secret named `AWS_ROLE_ARN`.
4. Set its value to the ARN of the IAM role trusted by the repository's GitHub OIDC provider.

The caller maps that repository secret into the reusable workflow:

```yaml
secrets:
  aws_role_arn: ${{ secrets.AWS_ROLE_ARN }}
```

The reusable workflow uses only the workflow-call secret:

```yaml
role-to-assume: ${{ secrets.aws_role_arn }}
```

The IAM role trust policy must allow GitHub's OIDC provider and restrict the permitted repository and branch or environment. The workflow uses the role in `us-east-1` for Terraform and ECR operations.
