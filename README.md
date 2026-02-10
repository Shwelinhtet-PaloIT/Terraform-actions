# Terraform-actions

GitHub composite actions for Terraform CI/CD pipelines with Azure. Designed for security-first DevOps workflows with OIDC authentication, multi-layer security scanning, and SARIF integration.

## Overview

This repository provides 11 modular, reusable composite actions that enable secure and efficient Terraform workflows on Azure. Each action follows security best practices, OIDC authentication (no static secrets), and  security scanning with SARIF uploads to GitHub Advanced Security.

**Key Features:**
-  **Security-First**: OIDC authentication, multi-layer scanning (tfsec, Checkov, Trivy), SARIF uploads
-  **Modular Design**: Compose actions into custom workflows matching your needs
-  **Production-Ready**: Pinned dependencies, error handling, grouped logging
-  **Azure-Optimized**: Native Azure backend support, ARM environment variables

## Quick Start

Basic PR validation workflow (no Azure authentication required):

```yaml
name: Terraform PR Validation
on: [pull_request]

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
        with:
          terraform-version: '1.7.0'
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
        with:
          working-directory: './terraform'
          enable-backend: 'false'
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-validate@a1b2c3d4 # v1.0.0
        with:
          working-directory: './terraform'
```

## Available Actions

| Action | Purpose | Key Feature |
|--------|---------|-------------|
| **[terraform-setup](docs/REFERENCE.md#terraform-setup)** | Foundation setup: checkout + Terraform install | Sets `TF_IN_AUTOMATION` env vars |
| **[azure-oidc-login](docs/REFERENCE.md#azure-oidc-login)** | Azure authentication via OIDC | Zero secrets, `ARM_*` env exports |
| **[terraform-init](docs/REFERENCE.md#terraform-init)** | Initialize Terraform working directory | Dual mode: backend on/off |
| **[terraform-validate](docs/REFERENCE.md#terraform-validate)** | Validate Terraform code quality | fmt + validate + tflint |
| **[determine-environment](docs/REFERENCE.md#determine-environment)** | Smart environment detection | Detects env from changed paths |
| **[tfplan-metadata](docs/REFERENCE.md#tfplan-metadata)** | Generate plan metadata and hash | SHA256 hash for integrity |
| **[scan-tfsec](docs/REFERENCE.md#scan-tfsec)** | tfsec Terraform security scan | SARIF upload to GitHub Security |
| **[scan-checkov](docs/REFERENCE.md#scan-checkov)** | Checkov policy-as-code scan | 500+ policies, SARIF upload |
| **[scan-trivy](docs/REFERENCE.md#scan-trivy)** | Trivy vulnerability scan | IaC config scanning |
| **[security-aggregate](docs/REFERENCE.md#security-aggregate)** | Combine security scan results | Single pass/fail determination |
| **[pr-comment-plan](docs/REFERENCE.md#pr-comment-plan)** | Post plan to PR comments | Auto-truncates large plans |

## Prerequisites

- **GitHub Actions knowledge**: Familiarity with workflows, jobs, and composite actions
- **Terraform experience**: Understanding of Terraform CLI and workflows
- **Azure OIDC setup**: Configured Azure service principal with Workload Identity Federation (for Azure backend operations)

## Documentation

- **[Action Reference](docs/REFERENCE.md)** - Complete specifications for all 11 actions (inputs, outputs, examples)
- **[Usage Examples](docs/EXAMPLES.md)** - Copy-paste workflow patterns for common scenarios

## License

Internal use only 
