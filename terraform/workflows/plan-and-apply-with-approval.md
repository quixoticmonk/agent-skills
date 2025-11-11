# Workflow: Plan and Apply with Approval

## Overview
This workflow implements a safe, human-in-the-loop Terraform deployment process that requires explicit approval before applying infrastructure changes. It's designed for production environments where change control is critical.

## Workflow Diagram

```
┌─────────────────┐
│  Code Change    │
│  Committed      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  terraform init │
│  Initialize     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ terraform plan  │
│ Generate Plan   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Plan Analysis  │
│  • Changes      │
│  • Resources    │
│  • Costs        │
└────────┬────────┘
         │
         ▼
    ┌────────┐
    │Approval│◄─────── Human Review
    │Request │         • Technical
    └────┬───┘         • Security
         │             • Cost
         │
    ┌────▼────┐
    │Approved?│
    └────┬────┘
         │
    ┌────┴────┐
    │         │
   Yes       No
    │         │
    │    ┌────▼────┐
    │    │  Stop   │
    │    │ Notify  │
    │    └─────────┘
    │
    ▼
┌─────────────────┐
│ terraform apply │
│ Execute Plan    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Verification   │
│  • Health       │
│  • Outputs      │
│  • Monitoring   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Notification   │
│  Success/Fail   │
└─────────────────┘
```

## Prerequisites

### Required Tools
- Terraform >= 1.5.0
- Git for version control
- CI/CD platform (GitHub Actions, GitLab CI, Jenkins, etc.)
- Notification system (Slack, Email, PagerDuty)

### Required Permissions
- Read/Write access to state backend
- Provider credentials (AWS, Azure, GCP)
- Approval permissions in CI/CD system

### Environment Setup
```bash
# Backend configuration
export TF_BACKEND_BUCKET="terraform-state-prod"
export TF_BACKEND_KEY="infrastructure/prod/terraform.tfstate"
export TF_BACKEND_REGION="us-east-1"

# Provider credentials
export AWS_PROFILE="production"
# or
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."

# Notification webhooks
export SLACK_WEBHOOK_URL="https://hooks.slack.com/..."
export APPROVAL_TEAM="@infrastructure-team"
```

## Workflow Steps

### Step 1: Initialize Terraform

```bash
#!/bin/bash
set -euo pipefail

echo "🔧 Initializing Terraform..."

terraform init \
  -backend-config="bucket=${TF_BACKEND_BUCKET}" \
  -backend-config="key=${TF_BACKEND_KEY}" \
  -backend-config="region=${TF_BACKEND_REGION}" \
  -upgrade

if [ $? -eq 0 ]; then
  echo "✅ Initialization successful"
else
  echo "❌ Initialization failed"
  exit 1
fi
```

### Step 2: Generate and Analyze Plan

```bash
#!/bin/bash
set -euo pipefail

echo "📋 Generating Terraform plan..."

# Create plan with detailed output
terraform plan \
  -out=tfplan \
  -detailed-exitcode \
  -lock=true \
  -input=false

PLAN_EXIT_CODE=$?

# Exit codes:
# 0 = No changes
# 1 = Error
# 2 = Changes present

case $PLAN_EXIT_CODE in
  0)
    echo "ℹ️  No changes detected"
    exit 0
    ;;
  1)
    echo "❌ Plan generation failed"
    exit 1
    ;;
  2)
    echo "📝 Changes detected, proceeding to analysis..."
    ;;
esac

# Generate human-readable plan
terraform show -no-color tfplan > plan_output.txt

# Generate JSON for programmatic analysis
terraform show -json tfplan > plan.json

echo "✅ Plan generated successfully"
```

### Step 3: Plan Analysis and Summary

```bash
#!/bin/bash
set -euo pipefail

echo "🔍 Analyzing plan..."

# Extract change statistics
ADDITIONS=$(jq -r '[.resource_changes[]? | select(.change.actions | contains(["create"]))] | length' plan.json)
CHANGES=$(jq -r '[.resource_changes[]? | select(.change.actions | contains(["update"]))] | length' plan.json)
DELETIONS=$(jq -r '[.resource_changes[]? | select(.change.actions | contains(["delete"]))] | length' plan.json)

# Identify high-risk changes
DESTRUCTIVE_CHANGES=$(jq -r '[.resource_changes[]? | select(.change.actions | contains(["delete", "create"]))] | length' plan.json)

# Generate summary
cat > plan_summary.md <<EOF
# Terraform Plan Summary

**Branch:** \`${GIT_BRANCH:-unknown}\`
**Commit:** \`${GIT_COMMIT:-unknown}\`
**Author:** ${GIT_AUTHOR:-unknown}

## Change Statistics
- **Resources to add:** ${ADDITIONS}
- **Resources to change:** ${CHANGES}
- **Resources to destroy:** ${DELETIONS}
- **Destructive changes (replace):** ${DESTRUCTIVE_CHANGES}

## Risk Assessment
$(if [ "$DESTRUCTIVE_CHANGES" -gt 0 ]; then
  echo "⚠️ **HIGH RISK**: Plan includes destructive changes (resource replacements)"
elif [ "$DELETIONS" -gt 0 ]; then
  echo "⚠️ **MEDIUM RISK**: Plan includes resource deletions"
elif [ "$CHANGES" -gt 0 ]; then
  echo "ℹ️ **LOW RISK**: Plan includes only updates to existing resources"
else
  echo "✅ **MINIMAL RISK**: Plan includes only new resources"
fi)

## Detailed Changes
\`\`\`
$(cat plan_output.txt | head -n 100)
...
(See full plan in artifacts)
\`\`\`

## Cost Impact
$(if command -v infracost &> /dev/null; then
  infracost breakdown --path plan.json --format markdown
else
  echo "_Cost estimation not available (infracost not installed)_"
fi)

## Security Scan
$(if command -v tfsec &> /dev/null; then
  tfsec . --format markdown --soft-fail || true
else
  echo "_Security scan not available (tfsec not installed)_"
fi)

## Next Steps
- Review the plan carefully
- Verify no unexpected changes
- Approve to proceed with apply
- Reject if changes are incorrect
EOF

cat plan_summary.md
```

### Step 4: Request Approval

#### GitHub Actions Example
```yaml
# .github/workflows/terraform-deploy.yml
name: Terraform Deploy

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read
  pull-requests: write
  deployments: write

jobs:
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.5.0
      
      - name: Terraform Init
        run: terraform init
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -out=tfplan -detailed-exitcode
          terraform show -no-color tfplan > plan_output.txt
        continue-on-error: true
      
      - name: Post Plan to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan_output.txt', 'utf8');
            const truncatedPlan = plan.length > 65000 
              ? plan.substring(0, 65000) + '\n\n... (truncated)'
              : plan;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan\n\`\`\`terraform\n${truncatedPlan}\n\`\`\``
            });
      
      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan
  
  apply:
    name: Terraform Apply
    needs: plan
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://console.aws.amazon.com
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
      
      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan
      
      - name: Terraform Init
        run: terraform init
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Notify Success
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-Type: application/json' \
            -d '{"text":"✅ Terraform apply successful for production"}'
      
      - name: Notify Failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-Type: application/json' \
            -d '{"text":"❌ Terraform apply failed for production"}'
```

#### GitLab CI Example
```yaml
# .gitlab-ci.yml
stages:
  - plan
  - approve
  - apply

variables:
  TF_ROOT: ${CI_PROJECT_DIR}
  TF_STATE_NAME: production

.terraform-base:
  image: hashicorp/terraform:1.5
  before_script:
    - cd ${TF_ROOT}
    - terraform init

plan:
  extends: .terraform-base
  stage: plan
  script:
    - terraform plan -out=tfplan
    - terraform show -no-color tfplan > plan_output.txt
  artifacts:
    paths:
      - tfplan
      - plan_output.txt
    expire_in: 1 week
  only:
    - main

apply:
  extends: .terraform-base
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan
  when: manual  # Requires manual approval
  only:
    - main
  environment:
    name: production
```

### Step 5: Apply Changes

```bash
#!/bin/bash
set -euo pipefail

echo "🚀 Applying Terraform changes..."

# Verify plan exists
if [ ! -f "tfplan" ]; then
  echo "❌ Plan file not found. Run terraform plan first."
  exit 1
fi

# Apply the plan
terraform apply \
  -auto-approve \
  -lock=true \
  tfplan

APPLY_EXIT_CODE=$?

if [ $APPLY_EXIT_CODE -eq 0 ]; then
  echo "✅ Apply successful"
  
  # Capture outputs
  terraform output -json > outputs.json
  
  # Clean up plan file
  rm -f tfplan
  
else
  echo "❌ Apply failed with exit code $APPLY_EXIT_CODE"
  exit $APPLY_EXIT_CODE
fi
```

### Step 6: Post-Apply Verification

```bash
#!/bin/bash
set -euo pipefail

echo "🔎 Verifying deployment..."

# Run validation
terraform validate

# Check state health
terraform state list > state_resources.txt
RESOURCE_COUNT=$(wc -l < state_resources.txt)
echo "ℹ️  Managing ${RESOURCE_COUNT} resources in state"

# Verify critical outputs
CRITICAL_OUTPUTS=("vpc_id" "load_balancer_dns" "database_endpoint")

for output in "${CRITICAL_OUTPUTS[@]}"; do
  value=$(terraform output -raw "$output" 2>/dev/null || echo "")
  if [ -z "$value" ]; then
    echo "⚠️  Warning: Critical output '$output' is empty"
  else
    echo "✅ Output '$output' = $value"
  fi
done

# Custom health checks
if [ -f "verify.sh" ]; then
  echo "Running custom verification script..."
  bash verify.sh
fi

echo "✅ Verification complete"
```

### Step 7: Notifications

```bash
#!/bin/bash

send_slack_notification() {
  local status=$1
  local message=$2
  
  local color
  case $status in
    success) color="good" ;;
    failure) color="danger" ;;
    *) color="warning" ;;
  esac
  
  curl -X POST "${SLACK_WEBHOOK_URL}" \
    -H 'Content-Type: application/json' \
    -d @- <<EOF
{
  "attachments": [{
    "color": "${color}",
    "title": "Terraform Deployment - ${status}",
    "fields": [
      {
        "title": "Environment",
        "value": "${ENVIRONMENT:-production}",
        "short": true
      },
      {
        "title": "Branch",
        "value": "${GIT_BRANCH:-main}",
        "short": true
      },
      {
        "title": "Commit",
        "value": "${GIT_COMMIT:-unknown}",
        "short": true
      },
      {
        "title": "Author",
        "value": "${GIT_AUTHOR:-unknown}",
        "short": true
      }
    ],
    "text": "${message}",
    "footer": "Terraform Automation",
    "ts": $(date +%s)
  }]
}
EOF
}

# Usage examples:
# send_slack_notification "success" "All resources deployed successfully"
# send_slack_notification "failure" "Deployment failed during apply phase"
```

## Approval Matrix

| Environment | Changes | Approvers Required | Review Time |
|-------------|---------|-------------------|-------------|
| Development | Any | 0 (auto-approve) | Immediate |
| Staging | Any | 1 (team member) | < 1 hour |
| Production | Add/Update | 1 (senior engineer) | < 4 hours |
| Production | Delete/Replace | 2 (senior + manager) | < 24 hours |

## Rollback Procedure

```bash
#!/bin/bash
set -euo pipefail

echo "⏮️  Initiating rollback..."

# Get previous state version
CURRENT_VERSION=$(terraform state pull | jq -r '.version')
PREVIOUS_VERSION=$((CURRENT_VERSION - 1))

# For S3 backend with versioning
aws s3api list-object-versions \
  --bucket "${TF_BACKEND_BUCKET}" \
  --prefix "${TF_BACKEND_KEY}" \
  --max-items 2

# Restore previous state (USE WITH CAUTION)
# This should be a last resort; prefer using git revert + new apply
echo "⚠️  Manual rollback required. Steps:"
echo "1. Revert git commit: git revert HEAD"
echo "2. Push revert: git push origin main"
echo "3. Re-run plan and apply workflow"
echo ""
echo "For emergency state restoration:"
echo "aws s3api get-object-version --bucket ... --key ... --version-id ..."
```

## Best Practices

### 1. Always Use Version Control
```bash
# Every change should be in git
git add .
git commit -m "feat: add new RDS instance"
git push origin feature/add-rds

# Create PR for review
gh pr create --title "Add RDS instance" --body "..."
```

### 2. Never Skip Plan
```bash
# ❌ Never do this in production
terraform apply -auto-approve

# ✅ Always plan first
terraform plan -out=tfplan
# Review plan
terraform apply tfplan
```

### 3. Use Remote State Locking
```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"  # Enable locking
    encrypt        = true
  }
}
```

### 4. Implement Plan Retention
```bash
# Save plans for audit trail
PLAN_DIR="plans/$(date +%Y-%m-%d_%H-%M-%S)"
mkdir -p "$PLAN_DIR"
cp tfplan "$PLAN_DIR/"
cp plan_output.txt "$PLAN_DIR/"
cp plan.json "$PLAN_DIR/"
```

### 5. Automate Security Scanning
```bash
# Run before approval
tfsec . --format json > tfsec-results.json
checkov -d . --output json > checkov-results.json
terrascan scan -o json > terrascan-results.json

# Block on critical findings
CRITICAL_COUNT=$(jq '[.results[] | select(.severity == "CRITICAL")] | length' tfsec-results.json)
if [ "$CRITICAL_COUNT" -gt 0 ]; then
  echo "❌ Critical security issues found. Blocking deployment."
  exit 1
fi
```

## Troubleshooting

### Issue: Plan Shows Unexpected Changes
```bash
# Compare with previous state
terraform show -json | jq '.values.root_module.resources' > current.json
aws s3 cp s3://${TF_BACKEND_BUCKET}/${TF_BACKEND_KEY} previous.tfstate
terraform show -json previous.tfstate | jq '.values.root_module.resources' > previous.json
diff previous.json current.json
```

### Issue: State Lock Conflict
```bash
# Check lock status
terraform force-unlock <LOCK_ID>

# For S3 backend with DynamoDB
aws dynamodb describe-table --table-name terraform-locks
aws dynamodb scan --table-name terraform-locks
```

### Issue: Apply Timeout
```bash
# Increase timeout for specific resources
resource "aws_db_instance" "main" {
  # ...
  
  timeouts {
    create = "60m"
    update = "60m"
    delete = "60m"
  }
}
```

## Monitoring and Alerting

```bash
# Track deployment metrics
cat > metrics.json <<EOF
{
  "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "duration_seconds": ${DURATION},
  "resources_added": ${ADDITIONS},
  "resources_changed": ${CHANGES},
  "resources_deleted": ${DELETIONS},
  "status": "${STATUS}"
}
EOF

# Send to monitoring system
curl -X POST https://metrics.example.com/terraform \
  -H 'Content-Type: application/json' \
  -d @metrics.json
```

## Related Workflows
- [Security Scan Workflow](./security-scan-workflow.md)
- [Disaster Recovery](./disaster-recovery.md)

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-11-07 | Initial workflow definition |
