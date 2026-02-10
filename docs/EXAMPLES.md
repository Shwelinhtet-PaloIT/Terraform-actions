# Workflow Examples

Complete workflow patterns for Terraform CI/CD. Copy-paste and adapt to your needs.

## Pattern 1: PR Validation

Validates Terraform code quality without Azure authentication.

**Permissions**: `contents: read`  
**Secrets**: None

```yaml
name: Terraform PR Validation

on:
  pull_request:
    branches: [main]
    paths: ['terraform/**']

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
          enable-tflint: 'true'
```

---

## Pattern 2: Security Scanning

Multi-scanner security validation with SARIF uploads to GitHub Security.

**Permissions**: `contents: read`, `security-events: write`  
**Secrets**: None

```yaml
name: Terraform Security Scanning

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # Weekly on Monday at 2 AM

permissions:
  contents: read
  security-events: write

jobs:
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
      # Step 1: Setup Terraform

permissions:
  contents: read
  security-events: write

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
        with:
          working-directory: './terraform'
          enable-backend: 'false'
      
      - id: tfsec
        uses: CDL-FirstOrg/Terraform-actions/actions/scan-tfsec@a1b2c3d4 # v1.0.0
        with:
          enabled: 'true'
          working-directory: './terraform'
          severity: 'HIGH'
          environment: 'dev'
      
      - id: checkov
        uses: CDL-FirstOrg/Terraform-actions/actions/scan-checkov@a1b2c3d4 # v1.0.0
        with:
          enabled: 'true'
          working-directory: './terraform'
          severity: 'HIGH'
          environment: 'dev'
      
      - id: trivy
        uses: CDL-FirstOrg/Terraform-actions/actions/scan-trivy@a1b2c3d4 # v1.0.0
        with:
          enabled: 'true'
          working-directory: './terraform'
          severity: 'HIGH'
          environment: 'dev'
      
      - if: always()
        uses: CDL-FirstOrg/Terraform-actions/actions/security-aggregate@a1b2c3d4 # v1.0.0
        with:
          tfsec-enabled: 'true'
          tfsec-outcome: ${{ steps.tfsec.outputs.outcome }}
          checkov-enabled: 'true'
          checkov-outcome: ${{ steps.checkov.outputs.outcome }}
          trivy-enabled: 'true'
          trivy-outcome: ${{ steps.trivy.outputs.outcome }}
```
  contents: read
  security-events: write
  id-token: write
  pull-requests: write
```

### Required Secrets

- `AZURE_CLIENT_ID` - Azure service principal client ID
- `AZURE_TENANT_ID` - Azure tenant ID
- `AZURE_SUBSCRIPTION_ID` - Azure subscription ID

### Complete Workflow

```yaml
name: Terraform Full CI Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write
  id-token: write
  pull-requests: write

env:
  TERRAFORM_VERSION: '1.7.0'
  WORKING_DIR: './terraform/environments/prod'

jobs:
  # Job 1: Validation
  validate:
    name: Validate Terraform
    runs-on: ubuntu-latest
    outputs:
      fmt-status: ${{ steps.validate.outputs.fmt-status }}
      validate-status: ${{ steps.validate.outputs.validate-status }}
      tflint-status: ${{ steps.validate.outputs.tflint-status }}
    
    steps:
      - name: Terraform Setup
        uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
        with:
          terraform-version: ${{ env.TERRAFORM_VERSION }}
      
      - name: Terraform Init
        uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
        with:
          working-directory: ${{ env.WORKING_DIR }}
          enable-backend: 'false'
      
      - name: Terraform Validate
        id: validate
        uses: CDL-FirstOrg/Terraform-actions/actions/terraform-validate@a1b2c3d4 # v1.0.0
        with:
          working-directory: ${{ env.WORKING_DIR }}
          enable-tflint: 'true'

  # Job 2: Security Scanning (parallel with validation)
  security:
    name: Security Scans
    runs-on: ubuntu-latest
    outputs:
      all-passed: ${{ steps.aggregate.outputs.all-passed }}
    
    steps:
      - name: Terraform Setup
        uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
        with:
          terraform-version: ${{ env.TERRAFORM_VERSION }}
      
      - name: Terraform Init
Complete pipeline with validation, security scanning, and plan generation with Azure OIDC.

**Permissions**: `contents: read`, `security-events: write`, `id-token: write`, `pull-requests: write`  
**Secrets**: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`: 'prod'
      
      - name: Run Trivy
        id: trivy
        uses: CDL-FirstOrg/Terraform-actions/actions/scan-trivy@a1b2c3d4 # v1.0.0
        with:
          enabled: 'true'
          working-directory: ${{ env.WORKING_DIR }}
          severity: 'HIGH'
          environment: 'prod'
      
      - name: Aggregate Results
        id: aggregate
        if: always()
        uses: CDL-FirstOrg/Terraform-actions/actions/security-aggregate@a1b2c3d4 # v1.0.0
        with:
          tfsec-enabled: 'true'
          tfsec-outcome: ${{ steps.tfsec.outputs.outcome }}
          checkov-enabled: 'true'
          checkov-outcome: ${{ steps.checkov.outputs.outcome }}
          trivy-enabled: 'true'
          trivy-outcome: ${{ steps.trivy.outputs.outcome }}

  # Job 3: Plan Generation (depends on validation & security)
  plan:
    name: Generate Terraform Plan

permissions:
  contents: read
  security-events: write
  id-token: write
  pull-requests: write

env:
  TF_VERSION: '1.7.0'
  WORKING_DIR: './terraform'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
        with:
          terraform-version: ${{ env.TF_VERSION }}
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
        with:
          working-directory: ${{ env.WORKING_DIR }}
          enable-backend: 'false'
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-validate@a1b2c3d4 # v1.0.0
        with:
          working-directory: ${{ env.WORKING_DIR }}

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
        with:
          working-directory: ${{ env.WORKING_DIR }}
          enable-backend: 'false'
      
      - id: tfsec
        uses: CDL-FirstOrg/Terraform-actions/actions/scan-tfsec@a1b2c3d4 # v1.0.0
        with:
          enabled: 'true'
          working-directory: ${{ env.WORKING_DIR }}
          severity: 'HIGH'
          environment: 'prod'
      
      - id: checkov
        uses: CDL-FirstOrg/Terraform-actions/actions/scan-checkov@a1b2c3d4 # v1.0.0
        with:
          enabled: 'true'
          working-directory: ${{ env.WORKING_DIR }}
          severity: 'HIGH'
          environment: 'prod'
      
      - id: trivy
        uses: CDL-FirstOrg/Terraform-actions/actions/scan-trivy@a1b2c3d4 # v1.0.0
        with:
          enabled: 'true'
          working-directory: ${{ env.WORKING_DIR }}
          severity: 'HIGH'
          environment: 'prod'
      
      - if: always()
        uses: CDL-FirstOrg/Terraform-actions/actions/security-aggregate@a1b2c3d4 # v1.0.0
        with:
          tfsec-enabled: 'true'
          tfsec-outcome: ${{ steps.tfsec.outputs.outcome }}
          checkov-enabled: 'true'
          checkov-outcome: ${{ steps.checkov.outputs.outcome }}
          trivy-enabled: 'true'
          trivy-outcome: ${{ steps.trivy.outputs.outcome }}

  plan:
    needs: [validate, security]
    runs-on: ubuntu-latest
    steps:
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-setup@a1b2c3d4 # v1.0.0
        with:
          terraform-version: ${{ env.TF_VERSION }}
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/azure-oidc-login@a1b2c3d4 # v1.0.0
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/terraform-init@a1b2c3d4 # v1.0.0
        with:
          working-directory: ${{ env.WORKING_DIR }}
          enable-backend: 'true'
      
      - id: plan
        run: terraform plan -out=tfplan -no-color | tee plan-output.txt
        working-directory: ${{ env.WORKING_DIR }}
      
      - id: metadata
        uses: CDL-FirstOrg/Terraform-actions/actions/tfplan-metadata@a1b2c3d4 # v1.0.0
        with:
          working-directory: ${{ env.WORKING_DIR }}
          environment: 'prod'
          terraform-version: ${{ env.TF_VERSION }}
      
      - uses: CDL-FirstOrg/Terraform-actions/actions/pr-comment-plan@a1b2c3d4 # v1.0.0
        with:
          working-directory: ${{ env.WORKING_DIR }}
          environment: 'prod'
          terraform-version: ${{ env.TF_VERSION }}
          plan-hash: ${{ steps.metadata.outputs.plan-hash }}
          plan-exitcode: '0'
          github-token: ${{ github.token }}
      
      - uses: actions/upload-artifact@v4
        with:
          name: ${{ steps.metadata.outputs.artifact-name }}
          path: |
            ${{ env.WORKING_DIR }}/tfplan
            ${{ env.WORKING_DIR }}/plan-output.txt
```

**Note**: Replace `@a1b2c3d4` with actual commit SHA or tag.