# Prompt: Summarize Terraform Plan

## Purpose
Generate a concise, human-readable summary of a Terraform plan that highlights key changes, risks, and resource impacts for review and approval.

## Input
A Terraform plan in JSON format (from `terraform show -json tfplan`)

## Output Format

```markdown
# Terraform Plan Summary

## Overview
- **Total Changes**: [X] resources
- **Actions**: [Y] to add, [Z] to change, [W] to destroy
- **Risk Level**: [CRITICAL/HIGH/MEDIUM/LOW]

## Resource Changes

### Resources to Add ([count])
- `aws_vpc.main` - VPC with CIDR 10.0.0.0/16
- `aws_subnet.public[0]` - Public subnet in us-east-1a
- ...

### Resources to Modify ([count])
- `aws_instance.web[0]` - Instance type: t3.micro → t3.small
- `aws_security_group.app` - Adding ingress rule for port 443
- ...

### Resources to Destroy ([count])
⚠️ **WARNING**: The following resources will be deleted:
- `aws_db_instance.legacy` - RDS database (potential data loss)
- `aws_s3_bucket.old_data` - S3 bucket (check for important data)
- ...

### Resources to Replace ([count])
🔄 **REPLACE** (destroy then create):
- `aws_launch_template.app` - Change requires replacement: ami_id
  - Triggers replacement of: aws_autoscaling_group.app
- ...

## Notable Changes

### Security Impact
- ✅ Enabling encryption on `aws_s3_bucket.data`
- ⚠️ Opening port 22 to 10.0.0.0/8 on `aws_security_group.web`
- ❌ Disabling versioning on `aws_s3_bucket.backups`

### Cost Impact
- **Estimated Monthly Change**: +$234.56
  - `aws_rds_instance.main`: +$150 (db.t3.micro → db.t3.medium)
  - `aws_instance.app`: +$84.56 (t3.small → t3.medium, count 1 → 3)

### Data Impact
- ⚠️ Potential data loss: `aws_db_instance.legacy` will be destroyed
- ℹ️ State migration: EBS volumes will be preserved via snapshot

## Dependency Chain
Notable resource dependencies:
```
aws_vpc.main
  └─ aws_subnet.public[*]
      └─ aws_instance.web[*]
          └─ aws_eip.web[*]
```

## Compliance Notes
- ✅ All new S3 buckets have encryption enabled
- ✅ RDS instances use encrypted storage
- ⚠️ Security group allows SSH from private network (requires approval)

## Risk Assessment

### Critical Risks
- Database instance replacement could cause downtime
- S3 bucket deletion without backup verification

### Mitigation Steps
1. Take manual snapshot of RDS before apply
2. Verify S3 bucket contents before deletion
3. Schedule maintenance window for database replacement

## Approval Checklist
- [ ] All changes reviewed and understood
- [ ] No unexpected resource deletions
- [ ] Data backup completed where necessary
- [ ] Compliance requirements satisfied
- [ ] Cost impact acceptable
- [ ] Stakeholders notified of changes

## Next Steps
1. Review detailed plan output
2. Verify changes align with intent
3. Approve or request modifications
4. Apply during scheduled maintenance window
```

## Implementation

```bash
#!/bin/bash

# Generate plan summary
terraform show -json tfplan | jq -r '
  .resource_changes as $changes |
  
  # Count changes by action
  ($changes | map(select(.change.actions | contains(["create"]))) | length) as $adds |
  ($changes | map(select(.change.actions | contains(["update"]))) | length) as $changes |
  ($changes | map(select(.change.actions | contains(["delete"]))) | length) as $deletes |
  ($changes | map(select(.change.actions | contains(["create", "delete"]))) | length) as $replaces |
  
  # Determine risk level
  (if $deletes > 0 or $replaces > 0 then "CRITICAL"
   elif $changes > 10 then "HIGH"
   elif $changes > 0 then "MEDIUM"
   else "LOW" end) as $risk |
  
  # Generate summary
  "# Terraform Plan Summary\n\n",
  "## Overview\n",
  "- **Total Changes**: \($adds + $changes + $deletes) resources\n",
  "- **Actions**: \($adds) to add, \($changes) to change, \($deletes) to destroy\n",
  "- **Risk Level**: \($risk)\n\n",
  
  # List resources by action
  "## Resource Changes\n\n",
  "### Resources to Add (\($adds))\n",
  ($changes | map(select(.change.actions | contains(["create"]))) | 
   map("- `\(.address)` - \(.type)") | join("\n")),
  "\n\n",
  
  "### Resources to Modify (\($changes))\n",
  ($changes | map(select(.change.actions | contains(["update"]))) | 
   map("- `\(.address)` - \(.type)") | join("\n")),
  "\n\n",
  
  "### Resources to Destroy (\($deletes))\n",
  "⚠️ **WARNING**: The following resources will be deleted:\n",
  ($changes | map(select(.change.actions | contains(["delete"]) and (contains(["create"]) | not))) | 
   map("- `\(.address)` - \(.type)") | join("\n")),
  "\n\n",
  
  "### Resources to Replace (\($replaces))\n",
  "🔄 **REPLACE** (destroy then create):\n",
  ($changes | map(select(.change.actions | contains(["create", "delete"]))) | 
   map("- `\(.address)` - \(.type)") | join("\n")),
  "\n"
'
```

## Usage in CI/CD

```yaml
# GitHub Actions
- name: Summarize Plan
  run: |
    terraform show -json tfplan > plan.json
    ./scripts/summarize-plan.sh plan.json > plan-summary.md
    
- name: Post Summary to PR
  uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      const summary = fs.readFileSync('plan-summary.md', 'utf8');
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: summary
      });
```

## Advanced Analysis

### Detect Sensitive Changes
```jq
# Find resources with sensitive attributes changing
.resource_changes[] | 
select(
  .change.after_unknown | 
  to_entries | 
  map(select(.key | test("password|secret|key"))) | 
  length > 0
) |
{
  address: .address,
  sensitive_fields: [.change.after_unknown | keys[] | select(test("password|secret|key"))]
}
```

### Calculate Resource Type Distribution
```jq
# Group changes by resource type
.resource_changes | 
group_by(.type) |
map({
  type: .[0].type,
  count: length,
  actions: map(.change.actions) | unique
})
```

### Identify Cross-Resource Impacts
```jq
# Find resources affected by replacements
.resource_changes[] |
select(.change.actions | contains(["create", "delete"])) |
{
  address: .address,
  affects: [
    .change.after_sensitive | 
    keys[] | 
    select(startswith("triggers_"))
  ]
}
```

## Customization

### Organization-Specific Rules
```bash
# Add custom checks for your organization
check_tagging_compliance() {
  local plan_json=$1
  
  # Ensure all resources have required tags
  jq -r '
    .resource_changes[] |
    select(.change.after.tags != null) |
    select(
      .change.after.tags | 
      has("Environment") and 
      has("Owner") and 
      has("CostCenter") |
      not
    ) |
    "⚠️ Missing required tags: \(.address)"
  ' "$plan_json"
}

# Check for specific naming patterns
check_naming_convention() {
  local plan_json=$1
  
  jq -r '
    .resource_changes[] |
    select(.change.after.name != null) |
    select(.change.after.name | test("^(dev|staging|prod)-") | not) |
    "⚠️ Name does not follow convention: \(.address) (\(.change.after.name))"
  ' "$plan_json"
}
```

## Integration with Approval Systems

### Slack Notification
```bash
send_plan_summary_to_slack() {
  local summary=$1
  local webhook=$2
  
  curl -X POST "$webhook" \
    -H 'Content-Type: application/json' \
    -d @- <<EOF
{
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "🔧 Terraform Plan Ready for Review"
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "$(cat "$summary" | jq -Rs .)"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {
            "type": "plain_text",
            "text": "✅ Approve"
          },
          "style": "primary",
          "url": "${APPROVAL_URL}"
        },
        {
          "type": "button",
          "text": {
            "type": "plain_text",
            "text": "❌ Reject"
          },
          "style": "danger",
          "url": "${REJECTION_URL}"
        }
      ]
    }
  ]
}
EOF
}
```

## Best Practices

1. **Always Include Context**: Branch, commit, author information
2. **Highlight Risks**: Make destructive changes obvious
3. **Provide Actions**: Clear next steps for reviewers
4. **Keep It Scannable**: Use formatting for quick review
5. **Include Cost Impact**: Help stakeholders make informed decisions
6. **Link to Details**: Provide access to full plan output
7. **Automate Generation**: Integrate into CI/CD pipeline

## Related Prompts
- [System Prompt](./system-prompt-tf.md)
- [Security Analysis](./analyze-security.md)

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-11-07 | Initial prompt definition |
