# GitHub Copilot Instructions for Terraform-actions

This repository provides 10 modular composite GitHub Actions for secure Terraform CI/CD on Azure.

## Architecture Overview

**Modular Composite Actions Pattern**: Each action in `actions/*/action.yml` is a standalone composite action that can be composed into workflows. Actions communicate via:
- **Environment variables** (`ARM_*`, `TF_*`) for cross-action state
- **Outputs** (`${{ steps.*.outputs.*}}`) for explicit data passing
- **File artifacts** (plan files, SARIF reports) for scan/plan results

**Security-First Design**: All actions use OIDC authentication (no static secrets), pinned dependencies (commit SHAs), and security scanning with SARIF uploads to GitHub Advanced Security.

## Critical Action Sequencing Rules

Actions MUST execute in this order:
```
1. terraform-setup       ← ALWAYS FIRST (checkout + Terraform install)
2. azure-oidc-login      ← Before terraform-init if Azure backend needed
3. terraform-init        ← Before ALL Terraform CLI commands
4. [validation/scanning] ← Can run in parallel
5. security-aggregate    ← After all scanners complete
6. tfplan-metadata       ← After terraform plan
7. pr-comment-plan       ← Final step for PR workflows
```

**Breaking these rules causes authentication/initialization failures.**

## Action YAML Structure Convention

All actions follow this consistent schema:
```yaml
name: 'Action Display Name'
description: 'Brief description'
inputs:
  input-name:
    description: 'Input description'
    required: true|false
    default: 'value'  # Only if not required
outputs:
  output-name:
    description: 'Output description'
    value: ${{ steps.step-id.outputs.value }}
runs:
  using: composite
  steps:
    - name: Step name
      id: step-id
      shell: bash
      run: |
        echo "::group::Grouped Output"
        # commands here
        echo "key=value" >> $GITHUB_OUTPUT
        echo "::endgroup::"
```

## Key Patterns to Follow

### 1. Dependency Pinning
**All external actions MUST use commit SHAs with version tags in comments:**
```yaml
uses: hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2
uses: azure/login@a457da9ea143d694b1b9c7c869ebb04ebe844ef5  # v2.3.0
```

### 2. Dual-Mode Actions
Several actions support enabled/disabled modes via boolean string inputs (`'true'`/`'false'`):
- `terraform-init`: `enable-backend: 'true'|'false'` (backend on/off)
- Security scanners: `enabled: 'true'|'false'`

**Always use string booleans**, not native YAML booleans.

### 3. Environment Variable Communication
Actions set env vars for downstream actions:
```yaml
- name: Export variables
  shell: bash
  run: |
    echo "ARM_CLIENT_ID=${{ inputs.client-id }}" >> $GITHUB_ENV
    echo "TF_IN_AUTOMATION=true" >> $GITHUB_ENV
```

### 4. Grouped Logging
Use `::group::` for organized output:
```bash
echo "::group::Section Name"
# commands
echo "::endgroup::"
```

### 5. Error Handling Pattern
```yaml
- name: Action step
  id: step
  continue-on-error: true  # For scanners/validation
  
- name: Evaluate result
  if: always()
  shell: bash
  run: |
    if [ "${{ steps.step.outcome }}" == "success" ]; then
      echo "✅ Passed"
    else
      echo "❌ Failed"
      exit 1
    fi
```

### 6. Security Aggregate Pattern
The `security-aggregate` action consolidates results from parallel scanners:
```yaml
# Run scanners in parallel
- uses: .../scan-tfsec
  id: tfsec
- uses: .../scan-checkov
  id: checkov
  
# Aggregate results
- uses: .../security-aggregate
  with:
    tfsec-enabled: 'true'
    tfsec-outcome: ${{ steps.tfsec.outputs.outcome }}
    # Similar for other scanners
```

## Reference Action by Usage Pattern

**Foundation actions** (always included):
- `terraform-setup`: Checkout code, install Terraform, set env vars
- `azure-oidc-login`: OIDC auth, exports `ARM_*` variables

**Operation actions**:
- `terraform-init`: Initialize with/without backend
- `terraform-validate`: fmt + validate + tflint checks
- `determine-environment`: Smart env detection from changed paths

**Security actions** (can run in parallel):
- `scan-tfsec`: tfsec security scan → SARIF
- `scan-checkov`: Checkov policy scan → SARIF
- `scan-trivy`: Trivy vulnerability scan → SARIF
- `security-aggregate`: Combine scan results

**Reporting actions**:
- `tfplan-metadata`: Generate plan hash and metadata JSON
- `pr-comment-plan`: Post plan output to PR comments

## Working with This Codebase

### Adding a New Action
1. Create `actions/action-name/action.yml`
2. Follow the YAML structure convention above
3. Pin all external action dependencies to commit SHAs
4. Use grouped logging for multi-step actions
5. Document in `docs/REFERENCE.md` with inputs/outputs table
6. Add usage example to `docs/EXAMPLES.md`

### Modifying Existing Actions
1. Maintain backward compatibility for inputs/outputs
2. Update pinned dependencies cautiously (test thoroughly)
3. Update corresponding `docs/REFERENCE.md` section
4. Preserve the action sequencing guarantees

### Testing Changes
Reference the patterns in `docs/EXAMPLES.md` for testing workflows:
- Pattern 1: PR validation (no Azure auth)
- Pattern 2: Security scanning
- Pattern 3: Full CI pipeline with OIDC

### Key Files Reference
- `actions/terraform-setup/action.yml`: Foundation pattern, always runs first
- `actions/azure-oidc-login/action.yml`: OIDC authentication pattern
- `actions/security-aggregate/action.yml`: Result aggregation pattern
- `actions/determine-environment/action.yml`: Complex path detection logic
- `docs/REFERENCE.md`: Complete action specifications
- `docs/EXAMPLES.md`: Full workflow examples
