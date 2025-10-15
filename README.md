# Shared Pipeline Blueprints

Central repository for reusable GitHub Actions workflows and composite actions for Terraform infrastructure provisioning across the organization.

## Overview

This repository provides a standardized, enterprise-grade Terraform CI/CD pipeline that includes:

- **Validation**: Terraform fmt, validate, tflint, and tfsec security scanning
- **Planning**: Generate and store Terraform execution plans
- **Security Scanning**: OPA policy validation and Trivy vulnerability scanning
- **Apply/Destroy**: Controlled infrastructure deployment and teardown
- **Multi-environment Support**: Parallel execution across DEV, SIT, UAT, PREPROD, and PROD

## Architecture

```
┌─────────────┐
│   GitHub    │
│ (main/feat) │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────────┐
│          Terraform Pipeline Workflow            │
├─────────────────────────────────────────────────┤
│  Validate → Plan → Scan → Apply/Destroy         │
│     ↓         ↓      ↓         ↓                │
│  Parallel Jobs Per Environment Workspace        │
│  (DEV, SIT, UAT, PREPROD, PROD)                 │
└─────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────┐
│     Remote State (AWS S3, Azure, GCS)           │
│     Identity (AWS IAM, Azure IAM, GCP IAM)      │
└─────────────────────────────────────────────────┘
```

## Repository Structure

```
.github/
├── actions/                        # Reusable composite actions
│   ├── assume-role/               # AWS/Azure/GCP role assumption
│   │   └── action.yml
│   ├── terraform-setup/           # Terraform version management
│   │   └── action.yml
│   └── terraform-init/            # Terraform initialization
│       └── action.yml
└── workflows/
    └── terraform-infrastructure-provisioning.yml  # Main pipeline workflow
```

## Workflows

### Terraform Infrastructure Provisioning

**File**: `.github/workflows/terraform-infrastructure-provisioning.yml`

**Triggers**:
- `push` to `main`, `feature/**`, `renovate/**` branches
- `pull_request` to `main`
- `workflow_dispatch` (manual trigger with options)

**Jobs**:

1. **Validate** - Runs on: push, PR, manual
   - Terraform format check
   - Terraform validate
   - TFLint static analysis
   - TFSec security scanning
   - Parallel execution per workspace
   - Uploads validation reports

2. **Plan** - Runs on: push, PR, manual (after Validate)
   - Generates Terraform execution plan
   - Parallel execution per workspace
   - Uploads plan artifacts

3. **Scan** - Runs on: push, PR, manual (after Plan)
   - OPA policy validation
   - Trivy security scanning
   - Parallel execution per workspace
   - Uploads scan results

4. **Apply** - Runs on: push to main, manual trigger (after Scan)
   - Applies Terraform plan
   - Sequential execution (max-parallel: 1)
   - Production safety controls

5. **Destroy** - Runs on: manual trigger only (after Scan)
   - Destroys infrastructure
   - Sequential execution (max-parallel: 1)
   - Requires explicit manual confirmation

## Composite Actions

### assume-role

Assumes cloud provider IAM roles for Terraform operations.

**Location**: `.github/actions/assume-role/action.yml`

**Inputs**:
- `role-arn` (required): IAM Role ARN to assume
- `external-id` (optional): External ID for role assumption
- `ignore-assume-role` (optional): Skip role assumption if 'true'

**Usage**:
```yaml
- name: Assume AWS Role
  uses: ./.github/actions/assume-role
  with:
    role-arn: arn:aws:iam::123456789012:role/TerraformRole
    ignore-assume-role: 'false'
```

### terraform-setup

Sets up Terraform version using tfenv or specified version.

**Location**: `.github/actions/terraform-setup/action.yml`

**Inputs**:
- `terraform-version` (optional): Terraform version to install
- `tf-current-source-dir` (optional): Terraform source directory

**Usage**:
```yaml
- name: Setup Terraform Environment
  uses: ./.github/actions/terraform-setup
  with:
    tf-current-source-dir: terraform
```

### terraform-init

Initializes Terraform with backend configuration and workspace selection.

**Location**: `.github/actions/terraform-init/action.yml`

**Inputs**:
- `tf-current-source-dir` (required): Terraform source directory
- `tf-variable-file` (optional): Path to Terraform variable file
- `workspace` (required): Terraform workspace name
- `backend-assume-role-arn` (optional): Backend IAM Role ARN

**Usage**:
```yaml
- name: Initialize Terraform
  uses: ./.github/actions/terraform-init
  with:
    tf-current-source-dir: terraform
    workspace: dev
    backend-assume-role-arn: arn:aws:iam::123456789012:role/TerraformBackendRole
```

## Usage in Other Repositories

### Option 1: Direct Workflow Reference (Recommended)

Create a workflow file in your repository that calls this reusable workflow:

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main, 'feature/**']
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  terraform:
    uses: datumhq-consulting-vn/bdo/shared-pipeline-blueprints/.github/workflows/terraform-infrastructure-provisioning.yml@main
    secrets: inherit
```

### Option 2: Copy and Customize

1. Copy the workflow and composite actions to your repository
2. Customize environment variables and workspace names
3. Adjust the Terraform source directory path

## Environment Configuration

### Required Repository Structure

Your Terraform repository should follow this structure:

```
your-terraform-repo/
├── terraform/                    # Terraform source directory
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── ...
├── environments/                 # Environment-specific variables
│   ├── dev.tfvars
│   ├── sit.tfvars
│   ├── uat.tfvars
│   ├── preprod.tfvars
│   └── prod.tfvars
└── .github/
    └── workflows/
        └── terraform.yml         # Calls shared pipeline
```

### Environment Variables

Configure these in your workflow or repository settings:

| Variable | Description | Default |
|----------|-------------|---------|
| `TF_CURRENT_SOURCE_DIR` | Terraform source directory | `terraform` |
| `VARIABLE_ROOT_FOLDER` | Environment variables folder | `environments` |
| `TF_BACKEND_ASSUME_ROLE_ARN` | Backend IAM Role ARN | - |
| `IGNORE_ASSUME_ROLE` | Skip role assumption | `false` |
| `MINIMUM_SEVERITY` | TFSec minimum severity | `MEDIUM` |
| `MINIMUM_FAILURE_SEVERITY` | TFLint failure severity | `warning` |

### Workspace Configuration

The pipeline runs against these workspaces by default:
- `dev` - Development environment
- `sit` - System Integration Testing
- `uat` - User Acceptance Testing
- `preprod` - Pre-production
- `prod` - Production

To customize workspaces, edit the `matrix.workspace` array in the workflow file.

### GitHub Environments

Configure GitHub Environments for each workspace:

1. Go to **Settings** → **Environments**
2. Create environments: `dev`, `sit`, `uat`, `preprod`, `prod`
3. Configure protection rules:
   - **prod/preprod**: Require reviewers, wait timer
   - **dev/sit/uat**: Optional protection

4. Add environment secrets:
   - Cloud provider credentials
   - Terraform backend configuration
   - Any workspace-specific secrets

## Runner Configuration

The workflows use `platform-shared-services` runner. Configure this in your GitHub organization:

1. Set up GitHub Actions self-hosted runners or runner scale sets
2. Label them with `platform-shared-services`
3. Ensure runners have access to:
   - Docker (for container jobs)
   - Cloud provider APIs
   - Terraform remote state storage

## Security Features

### Built-in Security Scanning

1. **TFSec**: Infrastructure-as-code security scanning
   - Scans for AWS, Azure, GCP misconfigurations
   - Generates JSON reports
   - Fails on CRITICAL severity issues

2. **TFLint**: Terraform linting
   - Detects syntax errors
   - Enforces best practices
   - Generates JUnit reports

3. **OPA (Open Policy Agent)**: Policy-as-code validation
   - Custom policy enforcement
   - Cost control policies
   - Compliance validation

4. **Trivy**: Vulnerability scanning
   - Scans Terraform configurations
   - Detects HIGH/CRITICAL vulnerabilities
   - Generates security reports

### Secret Management

- Never commit secrets to the repository
- Use GitHub Secrets for sensitive data
- Use cloud provider secret managers when possible
- Rotate credentials regularly

### Role-Based Access Control

- Use IAM roles with least privilege
- Implement cross-account role assumption
- Enable MFA for production deployments
- Audit all role assumptions

## Workflow Behavior

### On Push to main

```
Validate (parallel) → Plan (parallel) → Scan (parallel) → Apply (sequential)
  ├─ dev              ├─ dev            ├─ dev            └─ dev → sit → uat → preprod → prod
  ├─ sit              ├─ sit            ├─ sit
  ├─ uat              ├─ uat            ├─ uat
  ├─ preprod          ├─ preprod        ├─ preprod
  └─ prod             └─ prod           └─ prod
```

### On Pull Request

```
Validate (parallel) → Plan (parallel) → Scan (parallel)
  ├─ dev              ├─ dev            ├─ dev
  ├─ sit              ├─ sit            ├─ sit
  ├─ uat              ├─ uat            ├─ uat
  ├─ preprod          ├─ preprod        ├─ preprod
  └─ prod             └─ prod           └─ prod
```

### On Manual Trigger (workflow_dispatch)

Choose action:
- **validate**: Run validation only
- **plan**: Run validation + plan + scan
- **apply**: Run full pipeline including apply
- **destroy**: Run validation + plan + scan + destroy

## Artifact Management

Artifacts are stored with workspace-specific names:

| Artifact | Retention | Description |
|----------|-----------|-------------|
| `validate-reports-{workspace}` | 7 days | TFSec and TFLint reports |
| `terraform-plan-{workspace}` | 7 days | Terraform plan files |
| `scan-results-{workspace}` | 7 days | OPA and Trivy scan results |
| `apply-logs-{workspace}` | 30 days | Apply execution logs |
| `destroy-logs-{workspace}` | 30 days | Destroy execution logs |

## Troubleshooting

### Common Issues

**Issue**: Workflow doesn't trigger on push
- **Solution**: Check branch name matches trigger patterns (`main`, `feature/**`)

**Issue**: Matrix jobs fail with "workspace not found"
- **Solution**: Ensure tfvars files exist in `environments/` folder

**Issue**: Artifact download fails in downstream jobs
- **Solution**: Check artifact name matches between upload and download steps

**Issue**: Role assumption fails
- **Solution**: Verify IAM role trust relationship and permissions

**Issue**: Terraform init fails
- **Solution**: Check backend configuration and credentials

### Debug Mode

Enable debug logging:

1. Go to **Settings** → **Secrets**
2. Add `ACTIONS_STEP_DEBUG` = `true`
3. Add `ACTIONS_RUNNER_DEBUG` = `true`

## Best Practices

1. **Version Control**
   - Tag releases of this blueprint repository
   - Reference specific versions in consuming repos
   - Document breaking changes

2. **Testing**
   - Test changes in a fork first
   - Use feature branches for pipeline updates
   - Validate against multiple Terraform versions

3. **Monitoring**
   - Set up workflow failure notifications
   - Monitor artifact storage usage
   - Track workflow execution times

4. **Documentation**
   - Keep this README updated
   - Document custom policies
   - Maintain runbooks for common operations

## Contributing

To contribute to this shared pipeline:

1. Fork this repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request
6. Request review from platform team

## Support

For issues or questions:

- **GitHub Issues**: Create an issue in this repository
- **Slack**: #platform-engineering channel
- **Email**: platform-team@example.com

## License

Internal use only. All rights reserved.

## Changelog

### v1.0.0 (2025-01-15)
- Initial release
- Terraform validation, plan, scan, apply, destroy stages
- Multi-environment support (DEV, SIT, UAT, PREPROD, PROD)
- Composite actions for role assumption, setup, and init
- OPA and Trivy security scanning
- Artifact management with workspace isolation
