# Actions Reference

Complete reference for all 11 Terraform composite actions.

## Table of Contents

- [Action Sequencing Rules](#action-sequencing-rules)
- [Required Permissions](#required-permissions)
- [Foundation Actions](#foundation-actions)
  - [terraform-setup](#terraform-setup)
  - [azure-oidc-login](#azure-oidc-login)
- [Operations Actions](#operations-actions)
  - [terraform-init](#terraform-init)
  - [terraform-validate](#terraform-validate)
  - [determine-environment](#determine-environment)
  - [tfplan-metadata](#tfplan-metadata)
- [Security Actions](#security-actions)
  - [scan-tfsec](#scan-tfsec)
  - [scan-checkov](#scan-checkov)
  - [scan-trivy](#scan-trivy)
  - [security-aggregate](#security-aggregate)
- [Reporting Actions](#reporting-actions)
  - [pr-comment-plan](#pr-comment-plan)

---

## Action Sequencing Rules

Actions have dependencies and must be used in the correct order:

```
1. terraform-setup          ← ALWAYS FIRST (checkout + install Terraform)
2. azure-oidc-login         ← If Azure backend needed
3. terraform-init           ← After login (if used), before any terraform commands
4. terraform-validate       ← OR scan-* actions (can run in parallel)
   scan-tfsec
   scan-checkov
   scan-trivy
5. security-aggregate       ← After all scanners complete
6. tfplan-metadata          ← After `terraform plan` command
7. pr-comment-plan          ← Last step (PR comments)
```

**Critical Rules:**
- `terraform-setup` MUST be the first action in every job
- `azure-oidc-login` MUST come before `terraform-init` when using Azure backend
- `terraform-init` MUST come before any `terraform` CLI commands
- Security scanners (`scan-*`) can run in parallel but must all complete before `security-aggregate`

---

## Required Permissions

Different workflow patterns require different permissions:

| Pattern | Required Permissions |
|---------|---------------------|
| **PR Validation** | `contents: read` |
| **Security Scanning** | `contents: read`<br>`security-events: write` |
| **Plan Generation** | `contents: read`<br>`id-token: write` (OIDC)<br>`pull-requests: write` (comments) |
| **Full CI Pipeline** | `contents: read`<br>`security-events: write`<br>`id-token: write`<br>`pull-requests: write` |

---

## Foundation Actions

### terraform-setup

Checkout code, install Terraform, set environment variables. **Always run first.**

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `terraform-version` | No | `1.7.0` | Terraform version |

#### Outputs

| Name | Description |
|------|-------------|
| `terraform-version` | Installed version |

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
```

---

### azure-oidc-login

Azure OIDC authentication. Sets `ARM_*` environment variables.

#### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `client-id` | Yes | Azure client ID |
| `tenant-id` | Yes | Azure tenant ID |
| `subscription-id` | Yes | Azure subscription ID |

#### Outputs

None

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/azure-oidc-login@a1b2c3d4 # v1.0.0
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

## Operations Actions

### terraform-init

Initialize Terraform with optional backend.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `working-directory` | Yes | `.` | Terraform code directory |
| `enable-backend` | No | `true` | Enable backend (`'true'`/`'false'`) |

#### Outputs

None

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
  with:
    working-directory: './terraform'
    enable-backend: 'false'
```

---

### terraform-validate

Runs fmt, validate, and tflint checks.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `working-directory` | No | `.` | Terraform directory |
| `enable-tflint` | No | `true` | Enable tflint |
| `tflint-version` | No | `v0.60.0` | tflint version |

#### Outputs

| Name | Description |
|------|-------------|
| `fmt-status` | Format status |
| `validate-status` | Validate status |
| `tflint-status` | tflint status |

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/terraform-validate@a1b2c3d4 # v1.0.0
  with:
    working-directory: './terraform'
```

---

### determine-environment

Detects target environment from changed file paths.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `manual-environment` | No | `''` | Manual override |
| `working-directory-prefix` | No | `./env` | Directory prefix |
| `base-ref` | No | `main` | Base branch |
| `fallback-environment` | No | `dev` | Fallback env |

#### Outputs

| Name | Description |
|------|-------------|
| `environment` | Detected environment |
| `working-directory` | Full path |
| `changed-paths` | Changed paths |
| `detection-method` | Detection method |
| `is-multi-env` | Multiple envs changed |

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/determine-environment@a1b2c3d4 # v1.0.0
  id: env
```

---

### tfplan-metadata

Generate plan metadata and SHA256 hash.

#### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `working-directory` | Yes | Directory with tfplan |
| `environment` | Yes | Environment name |
| `terraform-version` | Yes | Terraform version |

#### Outputs

| Name | Description |
|------|-------------|
| `artifact-name` | Artifact name |
| `plan-hash` | SHA256 hash |

#### Example

```yaml
- run: terraform plan -out=tfplan
  working-directory: ./terraform

- uses: CDL-FirstOrg/Terraform-actions/actions/tfplan-metadata@a1b2c3d4 # v1.0.0
  id: metadata
  with:
    working-directory: './terraform'
    environment: 'prod'
    terraform-version: '1.7.0'
```

## Security Actions

### scan-tfsec

tfsec security scanner with SARIF upload.

#### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `enabled` | Yes | Enable scanner |
| `working-directory` | Yes | Terraform directory |
| `severity` | Yes | Min severity (`LOW`/`MEDIUM`/`HIGH`/`CRITICAL`) |
| `environment` | Yes | Environment name |
| `var-file` | No | Variable file |

#### Outputs

| Name | Description |
|------|-------------|
| `outcome` | `success`/`failure`/`skipped` |

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/scan-tfsec@a1b2c3d4 # v1.0.0
  with:
    enabled: 'true'
    working-directory: './terraform'
    severity: 'HIGH'
    environment: 'dev'
```

---

### scan-checkov

Checkov policy scanner with SARIF upload.

#### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `enabled` | Yes | Enable scanner |
| `working-directory` | Yes | Terraform directory |
| `severity` | Yes | Min severity |
| `environment` | Yes | Environment name |
| `var-file` | No | Variable file |

#### Outputs

| Name | Description |
|------|-------------|
| `outcome` | `success`/`failure`/`skipped` |

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/scan-checkov@a1b2c3d4 # v1.0.0
  with:
    enabled: 'true'
    working-directory: './terraform'
    severity: 'HIGH'
    environment: 'dev'
```

---

### scan-trivy

Trivy vulnerability scanner with SARIF upload.

#### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `enabled` | Yes | Enable scanner |
| `working-directory` | Yes | Terraform directory |
| `severity` | Yes | Min severity |
| `environment` | Yes | Environment name |

#### Outputs

| Name | Description |
|------|-------------|
| `outcome` | `success`/`failure`/`skipped` |

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/scan-trivy@a1b2c3d4 # v1.0.0
  with:
    enabled: 'true'
    working-directory: './terraform'
    severity: 'HIGH'
    environment: 'dev'
```

---

### security-aggregate

Aggregate security scanner results.

#### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `tfsec-enabled` | Yes | tfsec enabled |
| `tfsec-outcome` | Yes | tfsec outcome |
| `checkov-enabled` | Yes | Checkov enabled |
| `checkov-outcome` | Yes | Checkov outcome |
| `trivy-enabled` | Yes | Trivy enabled |
| `trivy-outcome` | Yes | Trivy outcome |

#### Outputs

| Name | Description |
|------|-------------|
| `all-passed` | All enabled scans passed |
| `tfsec-status` | tfsec status |
| `checkov-status` | Checkov status |
| `trivy-status` | Trivy status |

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/security-aggregate@a1b2c3d4 # v1.0.0
  with:
    tfsec-enabled: 'true'
    tfsec-outcome: ${{ steps.tfsec.outputs.outcome }}
    checkov-enabled: 'true'
    checkov-outcome: ${{ steps.checkov.outputs.outcome }}
    trivy-enabled: 'true'
    trivy-outcome: ${{ steps.trivy.outputs.outcome }}
```

---

## Reporting Actions

### pr-comment-plan

Post plan output to PR comments.

#### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `working-directory` | Yes | Directory with plan-output.txt |
| `environment` | Yes | Environment name |
| `terraform-version` | Yes | Terraform version |
| `plan-hash` | Yes | Plan hash from tfplan-metadata |
| `plan-exitcode` | Yes | Plan exit code |
| `github-token` | Yes | GitHub token |

#### Outputs

None

#### Example

```yaml
- run: terraform plan -no-color > plan-output.txt
  working-directory: ./terraform

- uses: CDL-FirstOrg/Terraform-actions/actions/pr-comment-plan@a1b2c3d4 # v1.0.0
  with:
    working-directory: './terraform'
    environment: 'dev'
    terraform-version: '1.7.0'
    plan-hash: ${{ steps.metadata.outputs.plan-hash }}
    plan-exitcode: '2'
    github-token: ${{ github.token }}
```
