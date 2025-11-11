# Prompt: Analyze Vault Radar Scan Results

## Purpose

This prompt guides AI agents to analyze Vault Radar scan results (JSON output) and provide actionable security insights with prioritized remediation recommendations.

## Input Format

Provide Vault Radar scan output in JSON format:

```json
{
  "scan_id": "scan-xyz789",
  "scan_timestamp": "2025-11-10T12:00:00Z",
  "scan_path": "/path/to/repo",
  "summary": {
    "total_events": 62,
    "by_severity": {...},
    "by_category": {...}
  },
  "events": [...]
}
```

Or provide event list from HCP Vault Radar:

```bash
vault-radar events list --output-format=json > events.json
```

## Analysis Tasks

When provided with scan results, perform:

### 1. Executive Summary

```markdown
# Vault Radar Security Analysis

**Repository**: [repo-name]
**Scan Date**: [timestamp]
**Total Findings**: [count]

## Risk Overview

| Severity | Count | Requires Immediate Action |
|----------|-------|---------------------------|
| Critical | X | ✅ Yes (< 1 hour) |
| High | X | ⚠️ Yes (< 24 hours) |
| Medium | X | Within 1 week |
| Low | X | Review and address |

## Risk Categories

- **Secrets**: X findings (Y active, Z inactive)
- **PII**: X findings
- **Non-Inclusive Language**: X findings
```

### 2. Critical Findings Detail

For each critical finding:

```markdown
### [Event ID]: [Secret Type]

**Severity**: Critical
**Status**: Active ⚠️
**Location**: `[file]:[line]`
**Exposure Duration**: [days/hours]
**Repository Visibility**: [public/private]

**Code Context**:
```[language]
[3 lines before]
→ [line with secret]  ← SECRET HERE
[3 lines after]
```

**Immediate Actions** (within 1 hour):
1. [Specific rotation command]
2. [Verification step]
3. [Migration to Vault/secrets manager]
4. [Audit log review]

**Estimated Remediation Time**: [X minutes]
**Blast Radius**: [What could be compromised]
```

### 3. Pattern Detection

Identify systemic issues:

```markdown
## Patterns Detected

### 🔴 Repeated Secret Types
- AWS Access Keys: 8 instances across 5 files
  - Suggests: Shared credentials instead of per-service keys
  - Recommendation: Use IAM roles or Vault AWS engine

### 🔴 Test Fixtures with Real Data
- 12 PII instances in tests/fixtures/
  - Risk: Test data exposure, compliance violations
  - Recommendation: Generate synthetic test data

### 🔴 Configuration Files
- 15 secrets in *.yml, *.json config files
  - Pattern: Configuration-as-code without secrets management
  - Recommendation: Externalize to Vault, use env vars injection
```

### 4. Prioritized Remediation Plan

```markdown
## Remediation Roadmap

### Phase 1: Immediate (Today)
**Critical secrets requiring rotation**

| Priority | Event ID | Type | Location | Action | Time |
|----------|----------|------|----------|--------|------|
| 1 | evt-001 | AWS Key | config.js:15 | Rotate + Vault | 20m |
| 2 | evt-002 | DB Pass | db.yml:8 | Vault DB engine | 30m |

**Commands**:
```bash
# evt-001: AWS Key rotation
aws iam delete-access-key --access-key-id AKIA...
vault write aws/config/root access_key=... secret_key=...
vault write aws/roles/myapp credential_type=iam_user ...
```

### Phase 2: Short-term (This Week)
**High severity + git history scrubbing**

- Scrub SSH keys from history (5 commits)
- Rotate API tokens (3 services)
- Implement pre-commit hooks

**Estimated effort**: 8 hours

### Phase 3: Medium-term (This Sprint)
**Medium severity + prevention**

- Review and update 18 medium severity findings
- Set up CI/CD scanning
- Enable Vault Radar GitHub App
- Developer training session

**Estimated effort**: 16 hours

### Phase 4: Cleanup (Next Sprint)
**Low severity + policy**

- Replace PII in test fixtures
- Update non-inclusive language
- Establish secrets baseline policy
- Quarterly audit schedule

**Estimated effort**: 8 hours
```

### 5. Compliance Impact Assessment

```markdown
## Compliance & Legal Considerations

### GDPR (if applicable)
- **PII Detected**: [count] instances
- **Citizen Data**: [EU resident data detected: YES/NO]
- **Breach Reporting**: [If exposed >72 hours, must report to DPA]
- **Actions**: Immediate PII removal + history scrubbing

### PCI DSS (if applicable)
- **Credit Card Data**: [count] instances
- **Compliance Status**: [FAIL - PCI DSS 3.2.1 Requirement 3.4]
- **Actions**: Immediate removal + security audit
- **Timeline**: Must remediate before next quarterly scan

### SOC 2 (if applicable)
- **Control Failure**: CC6.1 (Logical Access Controls)
- **Evidence**: Secrets in code = inadequate access controls
- **Impact**: Audit finding, potential certification risk
- **Actions**: Remediate + update controls documentation
```

### 6. Cost-Benefit Analysis

```markdown
## Remediation ROI

### Current Risk Exposure

**If AWS key compromised**:
- Potential AWS bill: $10K-$100K+ (crypto mining, data exfiltration)
- Data breach notification: $50K-$200K (legal, PR, customer notice)
- Compliance fines: $100K-$1M+ (GDPR, PCI DSS)
- Reputation damage: Immeasurable

**Total potential cost**: $160K - $1M+

### Remediation Investment

**Phase 1 (Critical)**: 2 hours engineering time (~$200)
**Phase 2-3 (Prevention)**: 24 hours engineering time (~$2,400)
**Annual maintenance**: 8 hours quarterly (~$3,200/year)

**Total year 1**: ~$5,800

### ROI Calculation

- **Risk reduction**: 95% (from high to minimal with prevention)
- **Avoided cost** (conservative): $160K × 0.95 = $152K
- **Investment**: $5,800
- **ROI**: 2,500% in year 1

**Recommendation**: Immediate approval for all remediation phases.
```

## Output Format

Provide structured markdown report with:

1. **Executive Summary** (1 page)
   - Risk overview table
   - Critical count requiring immediate action
   - High-level recommendations

2. **Critical Findings** (detailed)
   - One section per critical finding
   - Copy-paste ready remediation commands
   - Verification steps

3. **Remediation Roadmap** (tactical)
   - Phase-based plan with time estimates
   - Dependencies and prerequisites
   - Success criteria

4. **Prevention Strategy** (strategic)
   - Pre-commit hooks setup
   - CI/CD integration
   - Education and policy

5. **Appendix**
   - Full event list (if needed)
   - Compliance references
   - Tool installation guides

## Example Usage

### Input

```bash
# Scan repository
vault-radar scan repo --path=. --output-format=json > scan-results.json

# Provide to AI
cat scan-results.json | copilot analyze-scan
```

### AI Prompt

```
Using the prompt vault-radar/prompts/analyze-scan-results.md, analyze the 
attached scan-results.json and provide:

1. Executive summary of security posture
2. Detailed critical findings with remediation commands
3. Prioritized remediation roadmap
4. Estimated time and cost for each phase
5. Compliance impact assessment
```

### Expected Output

See sections 1-6 above for complete output structure.

## Customization

Adapt analysis based on:

### Organization Context

```markdown
**For startups**:
- Focus on speed: Quick wins, MVP prevention
- Lighter process: Pre-commit hooks, basic CI/CD
- Budget-conscious: Open source tools, Vault OSS

**For enterprises**:
- Comprehensive: Full compliance mapping
- Rigorous process: Multi-stage approval, change control
- Enterprise tools: Vault Enterprise, HCP premium features
```

### Industry-Specific

```markdown
**Financial Services**:
- Emphasize PCI DSS, SOC 2, GLBA compliance
- Highlight audit trail and controls
- Reference regulatory examination risk

**Healthcare**:
- HIPAA focus for PHI exposure
- Breach notification timelines (60 days)
- Reference OCR enforcement actions

**SaaS/Tech**:
- Customer trust and competitive positioning
- SOC 2 Type II certification risk
- Reference public breach disclosures
```

## Related Prompts

- `vault-radar/prompts/system-prompt-vault-radar.md` - Overall agent behavior
- `terraform/prompts/summarize-plan.md` - Similar analysis pattern for Terraform

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-11-10 | Initial prompt for scan analysis |
