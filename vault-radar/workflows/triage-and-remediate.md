# Workflow: Triage and Remediate Vault Radar Findings

## Overview

This workflow provides a systematic approach to handling secrets, PII, and non-inclusive language detected by HCP Vault Radar. It guides teams through assessment, prioritization, remediation, and verification while establishing preventive controls.

```
┌─────────────────────────────────────────────────────────────────┐
│                  Vault Radar Detection Event                     │
│                    (Secret, PII, or NIL)                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: Assess Severity                                         │
│  • Critical: Active secret in latest version → IMMEDIATE         │
│  • High: Active secret in old version → 24 hours                │
│  • Medium/Low: Review and schedule → 1 week                     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: Classify and Categorize                                │
│  • Secrets: Type (AWS, API key, etc.), Active status            │
│  • PII: Type (email, SSN, etc.), Context (prod vs test)         │
│  • NIL: Term identified, Replacement suggestion                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: Execute Remediation                                    │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │   Secrets   │  │     PII     │  │     NIL     │            │
│  │   Rotate    │  │   Remove    │  │   Replace   │            │
│  │   Migrate   │  │   Scrub     │  │   Update    │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 4: Scrub Git History (if needed)                          │
│  • Use git-filter-repo or BFG Repo-Cleaner                      │
│  • Coordinate with team for force push                          │
│  • Verify complete removal                                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 5: Verify Remediation                                     │
│  • Re-scan to confirm removal                                   │
│  • Verify secret rotation/deactivation                          │
│  • Check access logs for compromise                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 6: Implement Prevention                                   │
│  • Add pre-commit hooks                                         │
│  • Configure CI/CD scanning                                     │
│  • Update documentation and training                            │
│  • Mark finding as resolved in Vault Radar                      │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- HCP Vault Radar project with findings detected
- Access to affected repositories (git admin privileges for history scrubbing)
- Credentials/access to rotate secrets (AWS IAM, API providers, etc.)
- Vault Radar CLI installed for verification scans
- Team communication channel for coordination

## Step 1: Assess Severity and Impact

### 1.1 Retrieve Findings

```bash
# Get critical and high severity findings
vault-radar events list \
  --severity=critical,high \
  --output-format=json \
  --output-file=priority-findings.json

# Filter by risk category
vault-radar events list \
  --risk-category=secrets \
  --status=active \
  --limit=50
```

### 1.2 Severity Decision Matrix

| Severity | Response Time | Team Notification | Escalation |
|----------|---------------|-------------------|------------|
| Critical | Immediate (within 1 hour) | Security team + Dev lead | Incident response |
| High | Within 24 hours | Dev team | Security review |
| Medium | Within 1 week | Dev team | Sprint planning |
| Low | Within sprint | Async notification | Backlog |
| Info | Next maintenance window | Optional | Documentation |

### 1.3 Impact Assessment Questions

For each finding, determine:

- **Exposure Scope**: Is this in a public or private repository?
- **Time Window**: How long has the secret been exposed?
- **Access Frequency**: How many people/systems accessed this repository?
- **Active Usage**: Is the secret currently in use by production systems?
- **Blast Radius**: What systems/data could be compromised?

### 1.4 Create Incident Ticket

```markdown
# Vault Radar Finding: [Event ID]

**Severity**: Critical
**Risk Category**: Secret
**Secret Type**: AWS Access Key
**Status**: Active
**Detected**: 2025-11-09 14:22 UTC

## Location
- Repository: my-application
- File: `src/config/aws.js`
- Line: 15
- Commit: a1b2c3d4e5f6
- Branch: main

## Impact Assessment
- Exposure: Public repository
- Duration: ~30 days (since commit a1b2c3d4)
- Access: 15 developers + CI/CD systems
- Production: Yes, used in production deployments
- Blast Radius: Full AWS account access

## Immediate Actions
- [ ] Rotate AWS access key
- [ ] Audit CloudTrail logs
- [ ] Update application configuration
- [ ] Scrub from git history

## Assigned To
Security Team Lead: @security-lead
Developer: @app-owner

## SLA
Must be resolved within: 1 hour
```

## Step 2: Classify and Categorize

### 2.1 Secrets Classification

```bash
# Group secrets by type
vault-radar events list --risk-category=secrets --output-format=json | \
  jq 'group_by(.secret_type) | map({type: .[0].secret_type, count: length})'
```

**Common Secret Types**:

| Type | Rotation Method | Migration Target |
|------|----------------|------------------|
| AWS Access Keys | AWS IAM | Vault AWS secrets engine |
| GCP Service Accounts | GCP IAM | Vault GCP secrets engine |
| Database Passwords | Database admin | Vault database secrets engine |
| API Keys | Provider dashboard | Vault KV + auto-rotation |
| SSH Private Keys | ssh-keygen | Vault SSH secrets engine |
| TLS Certificates | Certificate Authority | Vault PKI secrets engine |

### 2.2 PII Classification

| PII Type | Remediation | Legal/Compliance |
|----------|-------------|------------------|
| Email addresses | Replace with synthetic data | GDPR, CCPA |
| SSN/National IDs | Remove completely | High compliance risk |
| Phone numbers | Replace with test patterns | Medium compliance risk |
| Credit cards | Remove completely | PCI DSS |
| Personal names | Anonymize or use placeholders | Context-dependent |

### 2.3 NIL Classification

| Term | Replacement | Context |
|------|-------------|---------|
| whitelist | allowlist | Access control |
| blacklist | denylist | Access control |
| master/slave | primary/replica, leader/follower | Databases, systems |
| grandfathered | legacy, pre-existing | Policies |
| man-hours | person-hours, work-hours | Estimation |

## Step 3: Execute Remediation

### 3.A: Remediate Secrets

#### Critical Priority (Active Secrets)

```bash
# Example: AWS Access Key Rotation

# 1. Create new access key
NEW_KEY=$(aws iam create-access-key --user-name myapp-user)
NEW_ACCESS_KEY=$(echo $NEW_KEY | jq -r '.AccessKey.AccessKeyId')
NEW_SECRET_KEY=$(echo $NEW_KEY | jq -r '.AccessKey.SecretAccessKey')

# 2. Update application configuration
# Option A: Vault (recommended)
vault kv put secret/app/aws \
  access_key=$NEW_ACCESS_KEY \
  secret_key=$NEW_SECRET_KEY

# Option B: AWS Secrets Manager
aws secretsmanager create-secret \
  --name /app/aws-credentials \
  --secret-string "{\"access_key\":\"$NEW_ACCESS_KEY\",\"secret_key\":\"$NEW_SECRET_KEY\"}"

# 3. Update application to read from Vault/Secrets Manager
# (See code examples below)

# 4. Verify new credentials work
aws sts get-caller-identity --profile new-key

# 5. Delete old access key
OLD_ACCESS_KEY="AKIAIOSFODNN7EXAMPLE"  # From Vault Radar finding
aws iam delete-access-key \
  --user-name myapp-user \
  --access-key-id $OLD_ACCESS_KEY

# 6. Audit access logs
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=$OLD_ACCESS_KEY \
  --max-results 100 \
  --output json > old-key-usage.json

# 7. Review for unauthorized access
jq '.Events[] | select(.Username != "myapp-user")' old-key-usage.json
```

**Application Update (Python with Vault)**:

```python
# Before (INSECURE):
AWS_ACCESS_KEY = "AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

s3_client = boto3.client(
    's3',
    aws_access_key_id=AWS_ACCESS_KEY,
    aws_secret_access_key=AWS_SECRET_KEY
)

# After (SECURE with Vault):
import hvac

vault_client = hvac.Client(url='https://vault.example.com')
vault_client.auth.approle.login(
    role_id=os.environ['VAULT_ROLE_ID'],
    secret_id=os.environ['VAULT_SECRET_ID']
)

aws_creds = vault_client.secrets.kv.v2.read_secret_version(
    path='app/aws',
    mount_point='secret'
)['data']['data']

s3_client = boto3.client(
    's3',
    aws_access_key_id=aws_creds['access_key'],
    aws_secret_access_key=aws_creds['secret_key']
)
```

#### Database Password Rotation

```bash
# 1. Enable Vault database secrets engine
vault secrets enable database

# 2. Configure PostgreSQL connection
vault write database/config/myapp-db \
  plugin_name=postgresql-database-plugin \
  allowed_roles="myapp-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.example.com:5432/myapp" \
  username="vault-admin" \
  password="current-password"

# 3. Create role for dynamic credentials
vault write database/roles/myapp-role \
  db_name=myapp-db \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# 4. Update application to use dynamic credentials
# (App requests new credentials from Vault on startup and periodically)

# 5. Rotate root credentials
vault write -force database/rotate-root/myapp-db

# 6. Verify old password no longer works
psql "postgresql://old-username:old-password@postgres.example.com:5432/myapp" -c '\l'
# Should fail with authentication error
```

### 3.B: Remediate PII

```bash
# 1. Identify PII locations
vault-radar events list --risk-category=pii --output-format=json > pii-findings.json

# 2. Review each finding
jq '.events[] | {file: .location.file_path, line: .location.line_number, type: .pii_type}' pii-findings.json

# 3. Replace with synthetic data
# Example: test fixtures with real emails
cat tests/fixtures/users.json
# Before:
# {"email": "john.doe@example.com", "phone": "555-123-4567"}

# After:
# {"email": "testuser1@example.test", "phone": "555-0100"}

# 4. For documentation/examples, use obviously fake data
# RFC 5737 reserved IPs: 192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24
# RFC 2606 reserved domains: .test, .example, .invalid, .localhost
# Example email: user@example.com
# Example SSN: 123-45-6789 (clearly fake pattern)
```

### 3.C: Remediate Non-Inclusive Language

```bash
# 1. Create replacement mapping
cat > nil-replacements.json <<EOF
{
  "whitelist": "allowlist",
  "blacklist": "denylist",
  "master": "main",
  "slave": "replica",
  "grandfathered": "legacy"
}
EOF

# 2. Use sed or automated tool for replacement
# Example: Update all occurrences
find . -type f -name "*.md" -o -name "*.go" -o -name "*.py" | \
  xargs sed -i.bak 's/whitelist/allowlist/g'

# 3. Manual review of context-sensitive terms
# Some terms may be acceptable in certain contexts (e.g., "master key" in cryptography)

# 4. Update documentation
# Add to style guide: "Use allowlist/denylist instead of whitelist/blacklist"
```

## Step 4: Scrub Git History

### 4.1 Using git-filter-repo (Recommended)

```bash
# Install git-filter-repo
# macOS:
brew install git-filter-repo
# Linux:
pip3 install git-filter-repo

# Backup repository
git clone --mirror git@github.com:org/repo.git repo-backup.git

# Remove specific file from all history
cd repo
git filter-repo --path config/secrets.yml --invert-paths

# Remove specific pattern from all files
git filter-repo --replace-text <(echo 'AKIAIOSFODNN7EXAMPLE==>***REMOVED***')

# Verify removal
git log --all --full-history --source -- config/secrets.yml
# Should return no results

# Force push (WARNING: Destructive operation)
git push origin --force --all
git push origin --force --tags
```

### 4.2 Using BFG Repo-Cleaner

```bash
# Install BFG
brew install bfg
# Or download from: https://rtyley.github.io/bfg-repo-cleaner/

# Clone mirror
git clone --mirror git@github.com:org/repo.git

# Remove secrets file
bfg --delete-files secrets.yml repo.git

# Remove specific text patterns
echo 'AKIAIOSFODNN7EXAMPLE' > secrets.txt
bfg --replace-text secrets.txt repo.git

# Clean up
cd repo.git
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Push
git push
```

### 4.3 Team Coordination

**Before scrubbing, communicate with team**:

```markdown
⚠️ IMPORTANT: Git History Rewrite Scheduled

We will be rewriting git history to remove exposed secrets on:
**Date**: November 11, 2025 at 10:00 AM UTC
**Repository**: my-application
**Reason**: Remove AWS credentials from commit history

## Action Required

All team members must:
1. Commit and push any pending work by 9:00 AM UTC
2. After the rewrite completes (~10:30 AM UTC):
   - Delete local clone: `rm -rf my-application`
   - Fresh clone: `git clone git@github.com:org/my-application.git`
3. DO NOT force push old branches

## Affected Branches
- main
- develop
- All feature branches

Questions? Contact: security-team@example.com
```

### 4.4 Verification

```bash
# After scrubbing, verify with Vault Radar
vault-radar scan repo --path=. --output-format=json > post-scrub-scan.json

# Check for the specific secret that was removed
jq '.events[] | select(.secret_type=="aws_access_key_id")' post-scrub-scan.json
# Should return empty array if successfully removed

# Verify git history
git log --all --source --full-history -S 'AKIAIOSFODNN7EXAMPLE'
# Should return no results
```

## Step 5: Verify Remediation

### 5.1 Re-scan Repository

```bash
# Full scan to confirm removal
vault-radar scan repo \
  --path=. \
  --output-format=json \
  --output-file=verification-scan.json

# Compare with original findings
# Check that specific event IDs are no longer present
```

### 5.2 Verify Secret Rotation

**AWS Access Key**:
```bash
# Attempt to use old key (should fail)
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE \
AWS_SECRET_ACCESS_KEY=old-secret \
aws sts get-caller-identity
# Expected: "An error occurred (InvalidClientTokenId)"

# Verify new key works
aws sts get-caller-identity --profile new-key
# Should succeed
```

**Database Password**:
```bash
# Old password should fail
psql "postgresql://myapp:old-password@db.example.com/myapp" -c '\l'
# Expected: authentication failed

# Application should successfully connect with Vault dynamic credentials
# Check application logs for successful database connections
```

### 5.3 Access Log Audit

```bash
# Review access patterns for signs of compromise
# AWS CloudTrail:
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --start-time 2025-11-01T00:00:00Z \
  --end-time 2025-11-10T23:59:59Z \
  --output json > access-audit.json

# Look for:
# - Unusual IP addresses
# - Unexpected API calls
# - Access from unknown regions
# - Off-hours activity

jq '.Events[] | {time: .EventTime, ip: .SourceIPAddress, event: .EventName}' access-audit.json | \
  sort | uniq -c | sort -rn
```

## Step 6: Implement Prevention

### 6.1 Pre-Commit Hooks

```bash
# Install pre-commit framework
pip install pre-commit

# Create .pre-commit-config.yaml
cat > .pre-commit-config.yaml <<EOF
repos:
  - repo: local
    hooks:
      - id: vault-radar-scan
        name: Vault Radar Secret Scan
        entry: scripts/vault-radar-precommit.sh
        language: script
        stages: [commit]
        pass_filenames: false
EOF

# Create scan script
cat > scripts/vault-radar-precommit.sh <<'EOF'
#!/bin/bash
set -e

echo "Running Vault Radar scan on staged files..."

# Get staged files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACM)

if [ -z "$STAGED_FILES" ]; then
  echo "No files to scan"
  exit 0
fi

# Create temp directory with staged content
TEMP_DIR=$(mktemp -d)
echo "$STAGED_FILES" | while read file; do
  mkdir -p "$TEMP_DIR/$(dirname "$file")"
  git show ":$file" > "$TEMP_DIR/$file" 2>/dev/null || true
done

# Scan
vault-radar scan repo \
  --path="$TEMP_DIR" \
  --output-format=json \
  --output-file=/tmp/vault-radar-precommit.json

# Check severity
CRITICAL=$(jq '[.events[] | select(.severity=="critical" or .severity=="high")] | length' /tmp/vault-radar-precommit.json)

if [ "$CRITICAL" -gt 0 ]; then
  echo "❌ Found $CRITICAL critical/high severity issues!"
  jq '.events[] | select(.severity=="critical" or .severity=="high") | {file: .location.file_path, line: .location.line_number, type: .secret_type}' /tmp/vault-radar-precommit.json
  rm -rf "$TEMP_DIR"
  exit 1
fi

echo "✅ Vault Radar scan passed"
rm -rf "$TEMP_DIR"
exit 0
EOF

chmod +x scripts/vault-radar-precommit.sh

# Install hooks
pre-commit install
```

### 6.2 CI/CD Integration

See `vault-radar/skills/scan-for-secrets/` for complete GitHub Actions and GitLab CI examples.

### 6.3 Vault Radar GitHub App

```bash
# Enable PR scanning via GitHub App
# 1. Go to HCP Vault Radar > Data Sources
# 2. Click "Add Data Source" > "GitHub App"
# 3. Install app on organization/repositories
# 4. Configure scan settings:
#    - Scan on PR: ✅
#    - Block PR on critical findings: ✅
#    - Comment on PR with results: ✅
```

### 6.4 Developer Training

**Topics to cover**:
- ✅ Never commit secrets to version control
- ✅ Use environment variables or secrets management
- ✅ Leverage Vault for dynamic credentials
- ✅ Run Vault Radar scans before pushing
- ✅ Understand severity levels and response times
- ✅ Use `.vault-radar-ignore` appropriately (not as a bypass)

### 6.5 Update Documentation

```markdown
# Developer Security Guidelines

## Secrets Management

### ❌ Never Do This
```python
# Hardcoded credentials
AWS_KEY = "AKIAIOSFODNN7EXAMPLE"
DB_PASSWORD = "MyS3cureP@ssw0rd"
```

### ✅ Always Do This
```python
import os
# Environment variables (for local development)
AWS_KEY = os.environ['AWS_ACCESS_KEY_ID']

# Or Vault (for production)
import hvac
vault = hvac.Client(url=os.environ['VAULT_ADDR'])
aws_creds = vault.secrets.kv.v2.read_secret_version(path='app/aws')
AWS_KEY = aws_creds['data']['data']['access_key']
```

## Pre-Commit Scanning

All commits are scanned by Vault Radar. To scan manually:

```bash
vault-radar scan repo --path=. --risk-categories=secrets
```

## What to Do If Secrets Are Committed

1. **DO NOT PUSH** - If not yet pushed, amend the commit
2. **If already pushed** - Contact security team immediately
3. **Follow** triage-and-remediate workflow
4. **Expected resolution time**: 1 hour for critical

## Resources

- Vault Radar Dashboard: https://portal.cloud.hashicorp.com/orgs/my-org/projects/my-project/vault-radar
- Secrets migration guide: `vault/workflows/migrate-to-vault.md`
- Questions? #security-team Slack channel
```

### 6.6 Mark as Resolved in Vault Radar

```bash
# After complete remediation
vault-radar events resolve \
  --event-id evt-abc123 \
  --resolution-note "Rotated AWS key, migrated to Vault, scrubbed git history, verified no unauthorized access"

# Bulk resolve (for systematic cleanup)
cat resolved-events.txt | while read event_id; do
  vault-radar events resolve --event-id "$event_id" --resolution-note "Batch remediation 2025-11-10"
done
```

## Best Practices

### Incident Response Coordination

1. **Communication**:
   - Create dedicated Slack/Teams channel for incident
   - Regular status updates (every 30 min for critical)
   - Clear ownership and escalation path

2. **Documentation**:
   - Keep timestamped notes of all actions taken
   - Save evidence (logs, scan results) for post-mortem
   - Document decisions and rationale

3. **Post-Incident Review**:
   ```markdown
   # Post-Incident Review: Secret Exposure evt-abc123
   
   **Date**: 2025-11-10
   **Severity**: Critical
   **Duration**: 45 minutes (detection to resolution)
   
   ## Timeline
   - 14:00 UTC: Secret detected by Vault Radar
   - 14:05 UTC: Security team notified
   - 14:10 UTC: Incident ticket created
   - 14:15 UTC: AWS key rotated
   - 14:30 UTC: Application updated, verified
   - 14:40 UTC: Git history scrubbed
   - 14:45 UTC: Verification scan clean
   
   ## What Went Well
   - Fast detection (< 5 minutes)
   - Clear runbook followed
   - No evidence of compromise in logs
   
   ## What Could Improve
   - Pre-commit hooks not installed on developer machine
   - Lack of automated secrets scanning in CI/CD
   - No alerting on sensitive file changes
   
   ## Action Items
   - [ ] Enforce pre-commit hooks via CI/CD check
   - [ ] Add Vault Radar to GitHub Actions
   - [ ] Create CODEOWNERS for sensitive files
   - [ ] Schedule quarterly secrets audit
   ```

### Prevention Over Remediation

**Cost Analysis**:
- Remediation time: 2-4 hours per incident (critical)
- Git history scrubbing: Additional 2-3 hours + coordination
- Risk of breach: Potentially millions in damages

**Prevention investment**:
- Pre-commit hooks setup: 30 minutes one-time
- CI/CD integration: 1 hour one-time
- Developer training: 1 hour quarterly
- ROI: 10x+ time savings, immeasurable risk reduction

## Common Pitfalls

### ❌ Marking as Resolved Without Rotation

**Problem**: Clicking "Resolve" in Vault Radar without actually rotating secret

**Impact**: Secret remains compromised

**Solution**: Checklist-driven remediation with verification step

### ❌ Incomplete Git History Scrubbing

**Problem**: Scrubbing only main branch, leaving feature branches

**Impact**: Secret accessible via branch history

**Solution**: Scrub all branches and tags, verify comprehensively

### ❌ Not Communicating Force Push

**Problem**: Force pushing without warning team

**Impact**: Team members lose work, confusion, duplicate effort

**Solution**: Clear communication 24 hours before, scheduled maintenance window

### ❌ Rotating to Another Hardcoded Secret

**Problem**: Replacing one hardcoded secret with another

**Impact**: Cycle repeats, no actual security improvement

**Solution**: Mandate migration to Vault or secrets manager, not just rotation

## Related Workflows

- `vault-radar/workflows/git-history-scrubbing.md` - Detailed git scrubbing procedures
- `vault/workflows/migrate-to-vault.md` - Moving secrets to Vault
- `vault/workflows/dynamic-database-credentials.md` - Implementing dynamic DB creds

## Related Skills

- `vault-radar/skills/scan-for-secrets/` - Running scans and interpreting results
- `vault-radar/skills/integrate-mcp-server/` - AI-assisted triage
- `vault/skills/generate-policy/` - Creating Vault policies for migrated secrets

## Resources

### Rotation Guides by Provider

- **AWS**: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html
- **GCP**: https://cloud.google.com/iam/docs/creating-managing-service-account-keys
- **Azure**: https://learn.microsoft.com/en-us/azure/key-vault/secrets/
- **GitHub**: https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token

### Git History Tools

- **git-filter-repo**: https://github.com/newren/git-filter-repo
- **BFG Repo-Cleaner**: https://rtyley.github.io/bfg-repo-cleaner/
- **GitHub secret scanning**: https://docs.github.com/en/code-security/secret-scanning

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-11-10 | Initial workflow creation |
