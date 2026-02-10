# Actions Reference

 To reference Terraform-actions composite actions.

## Table of Contents

- [Action Sequencing Rules](#action-sequencing-rules)
- [Required Permissions](#required-permissions)
- [Foundation Actions](#foundation-actions)
  - [terraform-setup](#terraform-setup)
  - [azure-oidc-login](#azure-oidc-login)
- [Operations Actions](#operations-actions)
  - [terraform-init](#terraform-init)
  - [terraform-validate](#terraform-validate)
  - [plan-metadata](#plan-metadata)
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
6. plan-metadata            ← After `terraform plan` command
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

Foundation action that checks out code, installs Terraform, and configures environment variables. **Always use this action first.**

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `terraform-version` | No | `1.7.0` | Terraform CLI version to install (e.g., `1.7.0`, `1.6.6`, `latest`) |

#### Outputs

| Name | Description |
|------|-------------|
| `terraform-version` | Installed Terraform version (extracted from `terraform version -json`) |

#### Environment Variables Set

- `TF_IN_AUTOMATION=true` - Terraform detects CI/CD environment
- `TF_INPUT=false` - Disable interactive prompts
- `TF_CLI_ARGS=-no-color` - Remove ANSI color codes from output

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
  with:
    terraform-version: '1.7.0'
```

#### Common Issues

- **Problem**: Checkout fails with authentication error
  - **Cause**: Private repository without proper access
  - **Solution**: Ensure workflow has `contents: read` permission or `GH_TOKEN` secret is set

---

### azure-oidc-login

Authenticate to Azure using OIDC (Workload Identity Federation) without static credentials. Sets `ARM_*` environment variables for Terraform Azure provider.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `client-id` | Yes | - | Azure service principal client ID |
| `tenant-id` | Yes | - | Azure tenant ID |
| `subscription-id` | Yes | - | Azure subscription ID |

#### Outputs

None (sets environment variables)

#### Environment Variables Set

- `ARM_CLIENT_ID` - Azure client ID for Terraform
- `ARM_TENANT_ID` - Azure tenant ID
- `ARM_SUBSCRIPTION_ID` - Azure subscription ID
- `ARM_USE_OIDC=true` - Enables OIDC authentication mode

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/azure-oidc-login@a1b2c3d4 # v1.0.0
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

#### Common Issues

- **Problem**: OIDC login fails with "Failed to login with federated token"
  - **Cause**: Workload Identity Federation not configured in Azure
  - **Solution**: Configure federated credentials in Azure App Registration linking to your GitHub repository
  
- **Problem**: Missing `id-token: write` permission
  - **Cause**: Workflow doesn't have permission to request OIDC token
  - **Solution**: Add `id-token: write` to workflow permissions block

---

## Operations Actions

### terraform-init

Initialize Terraform working directory with optional Azure backend configuration.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `working-directory` | Yes | `.` | Directory containing Terraform code |
| `enable-backend` | No | `true` | Enable Azure backend configuration (`true`/`false` string) |

#### Outputs

None

#### Backend Configuration Strategy

**`enable-backend: 'true'`** (default):
- Full initialization with Azure backend
- Requires `azure-oidc-login` before this action
- Reads `ARM_*` environment variables for backend authentication
- Use for: Plan generation, apply operations

**`enable-backend: 'false'`**:
- Local initialization without remote backend
- No Azure authentication required
- Backend state is not configured
- Use for: Security scans, PR validation checks

#### Example

```yaml
# For security scans (no Azure auth needed)
- uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
  with:
    working-directory: './terraform'
    enable-backend: 'false'

# For plan/apply (requires Azure auth)
- uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
  with:
    working-directory: './terraform'
    enable-backend: 'true'
```

#### Common Issues

- **Problem**: Backend initialization fails with authentication error
  - **Cause**: `azure-oidc-login` not run before this action, or credentials not exported
  - **Solution**: Ensure `azure-oidc-login` runs before `terraform-init` when `enable-backend: 'true'`

---

### terraform-validate

Run Terraform validation checks including format, validate, and tflint.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `working-directory` | No | `.` | Directory containing Terraform code |
| `enable-tflint` | No | `true` | Enable tflint code quality linting (`true`/`false` string) |
| `tflint-version` | No | `v0.60.0` | tflint version to use |

#### Outputs

| Name | Description |
|------|-------------|
| `fmt-status` | Format check status (`success`/`failure`) |
| `validate-status` | Validate status (`success`/`failure`) |
| `tflint-status` | tflint status (`success`/`failure`/`skipped`) |

#### Validation Layers

1. **terraform fmt** - Code formatting check (fails on formatting issues)
2. **terraform validate** - Syntax and configuration validation
3. **tflint** - Advanced linting and best practices (optional)

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/terraform-validate@a1b2c3d4 # v1.0.0
  with:
    working-directory: './terraform'
    enable-tflint: 'true'
    tflint-version: 'v0.60.0'
```

#### Common Issues

- **Problem**: Format check fails
  - **Cause**: Terraform files not formatted with `terraform fmt`
  - **Solution**: Run `terraform fmt -recursive` locally before commit

- **Problem**: tflint fails with module errors
  - **Cause**: tflint cannot resolve external modules
  - **Solution**: Ensure `terraform init` ran before validate, or disable tflint for complex modules

---

### plan-metadata

Generate metadata and SHA256 hash for Terraform plan artifacts. Use after `terraform plan -out=tfplan`.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `working-directory` | Yes | - | Directory containing `tfplan` file |
| `environment` | Yes | - | Target environment name (e.g., `dev`, `prod`) |
| `terraform-version` | Yes | - | Terraform version used for the plan |

#### Outputs

| Name | Description |
|------|-------------|
| `artifact-name` | Generated artifact name (format: `tfplan-{env}-{commit}-{timestamp}`) |
| `plan-hash` | SHA256 hash of the plan file for integrity verification |

#### Example

```yaml
- name: Generate Terraform Plan
  run: terraform plan -out=tfplan
  working-directory: ./terraform

- uses: CDL-FirstOrg/Terraform-actions/actions/tfplan-metadata@a1b2c3d4 # v1.0.0
  id: metadata
  with:
    working-directory: './terraform'
    environment: 'prod'
    terraform-version: '1.7.0'

- name: Use metadata
  run: |
    echo "Plan hash: ${{ steps.metadata.outputs.plan-hash }}"
    echo "Artifact: ${{ steps.metadata.outputs.artifact-name }}"
```

#### Common Issues

- **Problem**: Action fails with "tfplan file not found"
  - **Cause**: `terraform plan -out=tfplan` not run before this action
  - **Solution**: Ensure plan is generated in the same working directory

---

## Security Actions

### scan-tfsec

Run tfsec Terraform-specific security scanner with SARIF upload to GitHub Security.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `enabled` | Yes | - | Enable scanner (`'true'`/`'false'` string) |
| `working-directory` | Yes | - | Directory containing Terraform code |
| `severity` | Yes | - | Minimum severity to fail (`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`) |
| `environment` | Yes | - | Environment for SARIF category naming (e.g., `dev`) |
| `var-file` | No | `''` | Terraform variable file (optional) |

#### Outputs

| Name | Description |
|------|-------------|
| `outcome` | Scan result: `success`, `failure`, or `skipped` |

#### SARIF Category

Results uploaded to GitHub Security tab with category: `tfsec-{environment}`

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/scan-tfsec@a1b2c3d4 # v1.0.0
  with:
    enabled: 'true'
    working-directory: './terraform'
    severity: 'HIGH'
    environment: 'dev'
```

#### Common Issues

- **Problem**: SARIF upload fails
  - **Cause**: Missing `security-events: write` permission
  - **Solution**: Add permission to workflow permissions block

- **Problem**: Too many findings cause failure
  - **Cause**: Severity threshold too low
  - **Solution**: Start with `HIGH` or `CRITICAL`, lower as code improves

---

### scan-checkov

Run Checkov policy-as-code scanner with 500+ built-in policies and SARIF upload.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `enabled` | Yes | - | Enable scanner (`'true'`/`'false'` string) |
| `working-directory` | Yes | - | Directory containing Terraform code |
| `severity` | Yes | - | Minimum severity to fail (`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`) |
| `environment` | Yes | - | Environment for SARIF category naming (e.g., `dev`) |
| `var-file` | No | `''` | Terraform variable file (optional) |

#### Outputs

| Name | Description |
|------|-------------|
| `outcome` | Scan result: `success`, `failure`, or `skipped` |

#### SARIF Category

Results uploaded to GitHub Security tab with category: `checkov-{environment}`

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/scan-checkov@a1b2c3d4 # v1.0.0
  with:
    enabled: 'true'
    working-directory: './terraform'
    severity: 'HIGH'
    environment: 'dev'
```

#### Common Issues

- **Problem**: Checkov fails on external modules
  - **Cause**: `download_external_modules: false` prevents module download
  - **Solution**: Expected behavior - only local code is scanned for security

---

### scan-trivy

Run Trivy vulnerability scanner for IaC configuration scanning with SARIF upload.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `enabled` | Yes | - | Enable scanner (`'true'`/`'false'` string) |
| `working-directory` | Yes | - | Directory containing Terraform code |
| `severity` | Yes | - | Minimum severity to fail (`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`) |
| `environment` | Yes | - | Environment for SARIF category naming (e.g., `dev`) |

#### Outputs

| Name | Description |
|------|-------------|
| `outcome` | Scan result: `success`, `failure`, or `skipped` |

#### SARIF Category

Results uploaded to GitHub Security tab with category: `trivy-{environment}`

**Note**: Trivy uses different SARIF filename (`trivy-results.sarif`) to avoid conflicts.

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

Aggregate results from multiple security scanners and determine overall pass/fail status.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `tfsec-enabled` | Yes | - | Whether tfsec was enabled |
| `tfsec-outcome` | Yes | - | tfsec outcome from `scan-tfsec` output |
| `checkov-enabled` | Yes | - | Whether Checkov was enabled |
| `checkov-outcome` | Yes | - | Checkov outcome from `scan-checkov` output |
| `trivy-enabled` | Yes | - | Whether Trivy was enabled |
| `trivy-outcome` | Yes | - | Trivy outcome from `scan-trivy` output |

#### Outputs

| Name | Description |
|------|-------------|
| `all-passed` | Boolean: `true` if all enabled scanners passed, `false` otherwise |
| `tfsec-status` | Final tfsec status: `success`, `failure`, or `skipped` |
| `checkov-status` | Final Checkov status: `success`, `failure`, or `skipped` |
| `trivy-status` | Final Trivy status: `success`, `failure`, or `skipped` |

#### Aggregation Logic

- Only checks scanners that were **enabled**
- Skipped scanners do not affect overall result
- If any **enabled** scanner fails, `all-passed` is `false`
- All enabled scanners must pass for `all-passed` to be `true`

#### Example

```yaml
- uses: CDL-FirstOrg/Terraform-actions/actions/security-aggregate@a1b2c3d4 # v1.0.0
  id: aggregate
  with:
    tfsec-enabled: 'true'
    tfsec-outcome: ${{ steps.tfsec.outputs.outcome }}
    checkov-enabled: 'true'
    checkov-outcome: ${{ steps.checkov.outputs.outcome }}
    trivy-enabled: 'true'
    trivy-outcome: ${{ steps.trivy.outputs.outcome }}

- name: Check security results
  if: steps.aggregate.outputs.all-passed == 'false'
  run: exit 1
```

---

## Reporting Actions

### pr-comment-plan

Post Terraform plan output as a formatted comment on pull requests.

#### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `working-directory` | Yes | - | Directory containing `plan-output.txt` |
| `environment` | Yes | - | Target environment name |
| `terraform-version` | Yes | - | Terraform version used |
| `plan-hash` | Yes | - | SHA256 hash from `plan-metadata` |
| `plan-exitcode` | Yes | - | Terraform plan exit code (`0`=no changes, `2`=changes, other=error) |
| `github-token` | Yes | - | GitHub token (use `${{ github.token }}` or PAT) |

#### Outputs

None

#### Exit Code Interpretation

- `0` - No changes (infrastructure matches configuration)
- `2` - Changes detected (plan shows resource modifications)
- Other - Error occurred during planning

#### Example

```yaml
- name: Save plan output
  if: github.event_name == 'pull_request'
  run: terraform plan -no-color > plan-output.txt
  working-directory: ./terraform

- uses: CDL-FirstOrg/Terraform-actions/actions/pr-comment-plan@a1b2c3d4 # v1.0.0
  if: github.event_name == 'pull_request'
  with:
    working-directory: './terraform'
    environment: 'dev'
    terraform-version: '1.7.0'
    plan-hash: ${{ steps.metadata.outputs.plan-hash }}
    plan-exitcode: '2'
    github-token: ${{ github.token }}
```

#### Common Issues

- **Problem**: Comment not posted to PR
  - **Cause**: Not running on `pull_request` event, or missing `pull-requests: write` permission
  - **Solution**: Add permission and check event type with `if: github.event_name == 'pull_request'`

- **Problem**: Plan truncated in comment
  - **Cause**: Plan output exceeds 65,000 characters (GitHub limit)
  - **Solution**: Expected behavior - large plans are auto-truncated with indicator

---

## Additional Notes

### SARIF Category Naming

Each scanner uses a unique category (`{scanner}-{environment}`) to prevent result overwrites in GitHub Security tab. This allows multiple scans per environment to coexist.

**Example categories**:
- `tfsec-dev`
- `checkov-prod`
- `trivy-staging`

### Version Pinning

All examples use `@a1b2c3d4` (commit SHA) for security. Replace with your actual commit SHA or tag. Include version comment for clarity:

```yaml
uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@abc123def456 # v1.0.0
```
