# Workflow: Security Scanning for Terraform

## Overview
Automated security scanning workflow that identifies vulnerabilities, misconfigurations, and compliance violations in Terraform code before deployment. Integrates multiple security tools for comprehensive coverage.

## Security Tools Matrix

| Tool | Focus Area | Language | Speed | Best For |
|------|-----------|----------|-------|----------|
| **tfsec** | Security & compliance | HCL native | Fast | Quick security checks |
| **checkov** | Policy as code | Python | Medium | Multi-cloud, detailed |
| **terrascan** | Compliance frameworks | Go | Fast | OPA policies |
| **terraform-compliance** | BDD testing | Python | Medium | Custom requirements |
| **Sentinel** | Policy enforcement | Sentinel | Medium | Enterprise governance |

## Workflow Diagram

```
┌──────────────┐
│  Code Push   │
│  or PR       │
└──────┬───────┘
       │
       ▼
┌──────────────────────────┐
│  Pre-scan Validation     │
│  • terraform fmt -check  │
│  • terraform validate    │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│  Static Analysis         │
│  ├─ tfsec                │
│  ├─ checkov              │
│  └─ terrascan            │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│  Compliance Checks       │
│  • CIS Benchmarks        │
│  • PCI DSS               │
│  • HIPAA                 │
│  • SOC 2                 │
└──────┬───────────────────┘
       │
       ▼
┌──────────────────────────┐
│  Results Aggregation     │
│  • Severity ranking      │
│  • Deduplication         │
│  • Report generation     │
└──────┬───────────────────┘
       │
       ▼
    ┌──────┐
    │ Pass?│
    └──┬───┘
       │
   ┌───┴───┐
   │       │
  Yes      No
   │       │
   │   ┌───▼──────────┐
   │   │ Block Deploy │
   │   │ Create Issue │
   │   └──────────────┘
   │
   ▼
┌──────────────┐
│ Allow Deploy │
└──────────────┘
```

## Prerequisites

### Tool Installation
```bash
# Install tfsec
brew install tfsec
# or
curl -s https://raw.githubusercontent.com/aquasecurity/tfsec/master/scripts/install_linux.sh | bash

# Install checkov
pip install checkov

# Install terrascan
brew install terrascan
# or
curl -L "$(curl -s https://api.github.com/repos/tenable/terrascan/releases/latest | grep -o -E "https://.+?_Linux_x86_64.tar.gz")" > terrascan.tar.gz
tar -xf terrascan.tar.gz terrascan && rm terrascan.tar.gz
sudo mv terrascan /usr/local/bin

# Install terraform-compliance
pip install terraform-compliance
```

### Configuration Files

#### tfsec Custom Checks
```hcl
# .tfsec/config.yml
exclude:
  - AWS001  # Example: Exclude specific check if not applicable

severity_overrides:
  AWS002: ERROR  # Upgrade severity
  AWS003: WARNING  # Downgrade severity

minimum_severity: MEDIUM
```

#### checkov Configuration
```yaml
# .checkov.yml
framework: terraform
skip-check:
  - CKV_AWS_1  # Skip specific checks with justification

compact: false
quiet: false
output: json

soft-fail: false  # Fail on any finding
download-external-modules: true
```

#### terrascan Policy
```rego
# policies/custom/enforce_tags.rego
package accurics

# Ensure all AWS instances have required tags
requiredTags := ["Environment", "Owner", "CostCenter"]

deny[msg] {
  resource := input.aws_instance[name]
  not resource.config.tags
  msg := sprintf("AWS instance '%s' missing tags", [name])
}

deny[msg] {
  resource := input.aws_instance[name]
  existing_tags := {tag | resource.config.tags[tag]}
  required_tags := {tag | requiredTags[_] = tag}
  missing := required_tags - existing_tags
  count(missing) > 0
  msg := sprintf("Instance '%s' missing required tags: %v", [name, missing])
}
```

## Workflow Implementation

### Complete Scan Script
```bash
#!/bin/bash
set -euo pipefail

SCAN_DIR="${1:-.}"
REPORT_DIR="security-reports"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p "$REPORT_DIR"

echo "🔒 Starting security scan..."

# Exit codes
EXIT_CODE=0

# ===== FORMATTING CHECK =====
echo "📝 Checking code formatting..."
if ! terraform fmt -check -recursive "$SCAN_DIR"; then
  echo "❌ Code formatting issues found. Run 'terraform fmt -recursive' to fix."
  EXIT_CODE=1
fi

# ===== VALIDATION =====
echo "✅ Validating Terraform syntax..."
cd "$SCAN_DIR"
terraform init -backend=false > /dev/null 2>&1
if ! terraform validate; then
  echo "❌ Terraform validation failed"
  EXIT_CODE=1
fi
cd - > /dev/null

# ===== TFSEC SCAN =====
echo "🔍 Running tfsec scan..."
tfsec "$SCAN_DIR" \
  --format json \
  --out "$REPORT_DIR/tfsec-${TIMESTAMP}.json" \
  --soft-fail || TFSEC_EXIT=$?

tfsec "$SCAN_DIR" \
  --format default \
  --out "$REPORT_DIR/tfsec-${TIMESTAMP}.txt" \
  --soft-fail || true

# Parse results
TFSEC_CRITICAL=$(jq '[.results[] | select(.severity == "CRITICAL")] | length' "$REPORT_DIR/tfsec-${TIMESTAMP}.json")
TFSEC_HIGH=$(jq '[.results[] | select(.severity == "HIGH")] | length' "$REPORT_DIR/tfsec-${TIMESTAMP}.json")
TFSEC_MEDIUM=$(jq '[.results[] | select(.severity == "MEDIUM")] | length' "$REPORT_DIR/tfsec-${TIMESTAMP}.json")

echo "  Critical: $TFSEC_CRITICAL | High: $TFSEC_HIGH | Medium: $TFSEC_MEDIUM"

# ===== CHECKOV SCAN =====
echo "🔍 Running checkov scan..."
checkov -d "$SCAN_DIR" \
  --framework terraform \
  --output json \
  --output-file-path "$REPORT_DIR" \
  --soft-fail || CHECKOV_EXIT=$?

mv "$REPORT_DIR/results_json.json" "$REPORT_DIR/checkov-${TIMESTAMP}.json" 2>/dev/null || true

# Parse results
CHECKOV_FAILED=$(jq '.summary.failed' "$REPORT_DIR/checkov-${TIMESTAMP}.json" 2>/dev/null || echo "0")
CHECKOV_PASSED=$(jq '.summary.passed' "$REPORT_DIR/checkov-${TIMESTAMP}.json" 2>/dev/null || echo "0")

echo "  Failed: $CHECKOV_FAILED | Passed: $CHECKOV_PASSED"

# ===== TERRASCAN =====
echo "🔍 Running terrascan..."
terrascan scan \
  -d "$SCAN_DIR" \
  -o json \
  > "$REPORT_DIR/terrascan-${TIMESTAMP}.json" || TERRASCAN_EXIT=$?

terrascan scan \
  -d "$SCAN_DIR" \
  -o human \
  > "$REPORT_DIR/terrascan-${TIMESTAMP}.txt" || true

# Parse results
TERRASCAN_VIOLATIONS=$(jq '.results.violations | length' "$REPORT_DIR/terrascan-${TIMESTAMP}.json" 2>/dev/null || echo "0")

echo "  Violations: $TERRASCAN_VIOLATIONS"

# ===== AGGREGATE RESULTS =====
echo "📊 Generating summary report..."

cat > "$REPORT_DIR/summary-${TIMESTAMP}.md" <<EOF
# Security Scan Summary

**Scan Date:** $(date)
**Directory:** $SCAN_DIR

## Results Overview

| Tool | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| tfsec | $TFSEC_CRITICAL | $TFSEC_HIGH | $TFSEC_MEDIUM | - | $((TFSEC_CRITICAL + TFSEC_HIGH + TFSEC_MEDIUM)) |
| checkov | - | - | - | - | $CHECKOV_FAILED |
| terrascan | - | - | - | - | $TERRASCAN_VIOLATIONS |

## Detailed Findings

### tfsec Results
$(cat "$REPORT_DIR/tfsec-${TIMESTAMP}.txt" | head -n 50)

### checkov Results
$(jq -r '.results.failed_checks[] | "- [\(.severity)] \(.check_id): \(.name)"' "$REPORT_DIR/checkov-${TIMESTAMP}.json" 2>/dev/null | head -n 30 || echo "No failed checks")

### terrascan Results
$(jq -r '.results.violations[] | "- [\(.severity)] \(.rule_name): \(.description)"' "$REPORT_DIR/terrascan-${TIMESTAMP}.json" 2>/dev/null | head -n 30 || echo "No violations")

## Recommended Actions

$(if [ "$TFSEC_CRITICAL" -gt 0 ]; then
  echo "🚨 **CRITICAL**: Address critical tfsec findings immediately"
fi)

$(if [ "$TFSEC_HIGH" -gt 0 ]; then
  echo "⚠️ **HIGH**: Review and remediate high-severity issues before deployment"
fi)

$(if [ "$CHECKOV_FAILED" -gt 10 ]; then
  echo "⚠️ **ATTENTION**: Large number of checkov failures detected"
fi)

## Compliance Status

$(check_compliance)

EOF

cat "$REPORT_DIR/summary-${TIMESTAMP}.md"

# ===== POLICY ENFORCEMENT =====
echo "🚦 Applying policy enforcement..."

# Block on critical findings
if [ "$TFSEC_CRITICAL" -gt 0 ]; then
  echo "❌ CRITICAL security issues found. Deployment blocked."
  EXIT_CODE=1
fi

# Warn on high findings
if [ "$TFSEC_HIGH" -gt 5 ]; then
  echo "⚠️  WARNING: $TFSEC_HIGH high-severity issues found."
  # Optionally block: EXIT_CODE=1
fi

exit $EXIT_CODE
```

### Compliance Check Function
```bash
check_compliance() {
  local cis_compliant=true
  local pci_compliant=true
  
  # Check CIS AWS Foundations Benchmark
  local cis_checks=(
    "CKV_AWS_20"  # S3 versioning
    "CKV_AWS_21"  # S3 encryption
    "CKV_AWS_19"  # S3 access logging
  )
  
  for check in "${cis_checks[@]}"; do
    if jq -e ".results.failed_checks[] | select(.check_id == \"$check\")" \
       "$REPORT_DIR/checkov-${TIMESTAMP}.json" > /dev/null 2>&1; then
      cis_compliant=false
      break
    fi
  done
  
  if [ "$cis_compliant" = true ]; then
    echo "✅ **CIS AWS Foundations**: Compliant"
  else
    echo "❌ **CIS AWS Foundations**: Non-compliant"
  fi
  
  # Add similar checks for PCI, HIPAA, etc.
}
```

### CI/CD Integration

#### GitHub Actions
```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'
  push:
    branches: [main]

jobs:
  security-scan:
    name: Terraform Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: false
          format: sarif
          
      - name: Upload tfsec SARIF
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: tfsec.sarif
      
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          output_format: sarif
          soft_fail: false
          
      - name: Run Terrascan
        uses: tenable/terrascan-action@main
        with:
          iac_type: 'terraform'
          policy_type: 'aws'
          sarif_upload: true
          
      - name: Generate Security Report
        if: always()
        run: |
          ./scripts/generate-security-report.sh > security-report.md
          
      - name: Comment PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('security-report.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: report
            });
```

### Custom Security Policies

#### Terraform Compliance BDD Tests
```gherkin
# compliance/aws-security.feature
Feature: AWS Security Best Practices
  As a security engineer
  I want to ensure all AWS resources follow security best practices
  
  Scenario: S3 buckets must have encryption enabled
    Given I have aws_s3_bucket defined
    Then it must have aws_s3_bucket_server_side_encryption_configuration
    
  Scenario: S3 buckets must block public access
    Given I have aws_s3_bucket defined
    Then it must have aws_s3_bucket_public_access_block
    And it must contain block_public_acls
    And its value must be true
    
  Scenario: RDS instances must have encryption at rest
    Given I have aws_db_instance defined
    Then it must contain storage_encrypted
    And its value must be true
    
  Scenario: Security groups must not allow unrestricted ingress
    Given I have aws_security_group defined
    When it contains ingress
    Then it must not have cidr_blocks
    Or its value must not contain "0.0.0.0/0"
    
  Scenario: EC2 instances must have detailed monitoring
    Given I have aws_instance defined
    Then it must contain monitoring
    And its value must be true
    
  Scenario: All resources must have mandatory tags
    Given I have any resource defined
    When it supports tags
    Then it must contain tags
    And it must contain Environment
    And it must contain Owner
    And it must contain CostCenter
```

Run compliance tests:
```bash
terraform-compliance -p . -f compliance/aws-security.feature
```

## Security Finding Categories

### 1. Encryption Issues
```bash
# Detect unencrypted resources
grep -r "encrypted.*false" .
grep -r "kms_key_id" . | grep -v "aws_kms"
```

### 2. Network Exposure
```bash
# Find publicly accessible resources
grep -r "0.0.0.0/0" .
grep -r "publicly_accessible.*true" .
grep -r "map_public_ip_on_launch.*true" .
```

### 3. Access Control
```bash
# Identify overly permissive policies
grep -r '"Effect".*"Allow"' .
grep -r '"Action".*"\*"' .
grep -r '"Resource".*"\*"' .
```

### 4. Logging and Monitoring
```bash
# Check for missing logging
grep -L "logging" **/*.tf
grep -L "enable_logging" **/*.tf
```

## Remediation Patterns

### Fix: Enable S3 Encryption
```hcl
# Before (insecure)
resource "aws_s3_bucket" "data" {
  bucket = "my-bucket"
}

# After (secure)
resource "aws_s3_bucket" "data" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Fix: Restrict Security Group
```hcl
# Before (insecure)
resource "aws_security_group" "web" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # ❌ Open to internet
  }
}

# After (secure)
resource "aws_security_group" "web" {
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]  # ✅ Restricted to admin network
    description = "SSH access from admin network only"
  }
}
```

## Reporting

### HTML Report Generation
```bash
#!/bin/bash

# Generate comprehensive HTML report
cat > security-report.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
  <title>Terraform Security Scan Report</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; }
    .critical { background-color: #ff4444; color: white; }
    .high { background-color: #ff9944; color: white; }
    .medium { background-color: #ffdd44; }
    .low { background-color: #dddddd; }
    table { border-collapse: collapse; width: 100%; }
    th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
    th { background-color: #4CAF50; color: white; }
  </style>
</head>
<body>
  <h1>Terraform Security Scan Report</h1>
  <p>Generated: $(date)</p>
  
  <h2>Summary</h2>
  <table>
    <tr>
      <th>Tool</th>
      <th>Critical</th>
      <th>High</th>
      <th>Medium</th>
      <th>Low</th>
    </tr>
    <tr>
      <td>tfsec</td>
      <td class="critical">$TFSEC_CRITICAL</td>
      <td class="high">$TFSEC_HIGH</td>
      <td class="medium">$TFSEC_MEDIUM</td>
      <td class="low">-</td>
    </tr>
  </table>
  
  <h2>Detailed Findings</h2>
  <!-- Parse JSON and insert findings -->
  
</body>
</html>
EOF
```

## Best Practices

1. **Scan Early and Often**: Run on every commit and PR
2. **Fail Fast**: Block critical issues immediately
3. **Track Trends**: Monitor security posture over time
4. **Automate Remediation**: Use automated fixes where possible
5. **Custom Policies**: Implement organization-specific rules
6. **Regular Updates**: Keep scanning tools updated
7. **Developer Training**: Educate team on secure patterns

## Related Workflows
- [Plan and Apply with Approval](./plan-and-apply-with-approval.md)
- [Infrastructure Testing](./infrastructure-testing.md)

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-11-07 | Initial workflow definition |
