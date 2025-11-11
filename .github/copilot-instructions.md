# GitHub Copilot Instructions - HashiCorp Stack

## Overview
This repository contains curated instruction sets, skills, and workflows for AI agents working with HashiCorp products: Terraform, Vault, Vault Radar, and Consul. Use these instructions to enhance AI-assisted development with context-aware, security-focused guidance.

## How to Use This Library

### With GitHub Copilot Chat
Reference specific skills or workflows in your prompts:

```
@workspace Using the Terraform generate-hcl skill, create an AWS VPC module
```

```
@workspace Following the Vault generate-policy skill, create a policy for my application
```

```
@workspace Apply the plan-and-apply-with-approval workflow for this change
```

### With Agent Mode
When using AI agents for autonomous development:

1. **Load System Prompt**: Start with product-specific system prompts
   - `/terraform/prompts/system-prompt-tf.md` for Terraform work
   - `/vault/prompts/system-prompt-vault.md` for Vault configurations

2. **Reference Skills**: Point agents to relevant skills for specific tasks
   - Code generation → `/terraform/skills/generate-hcl/`
   - Policy creation → `/vault/skills/generate-policy/`
   - Refactoring → `/terraform/skills/refactor-module/`
   - Secrets scanning → `/vault-radar/skills/scan-for-secrets/`
   - MCP integration → `/vault-radar/skills/integrate-mcp-server/`

3. **Follow Workflows**: Apply workflows for complex processes
   - `/terraform/workflows/plan-and-apply-with-approval.md`
   - `/terraform/workflows/security-scan-workflow.md`
   - `/vault/workflows/new-kv-engine-setup.md`
   - `/vault-radar/workflows/triage-and-remediate.md`

## Product-Specific Guidance

### Terraform
When working with Terraform configurations:
- Always query Terraform Registry for latest versions before code generation
- Follow HCL best practices from `/terraform/skills/generate-hcl/resources/`
- Apply security scanning before deployment
- Use explicit typing and validation for all variables
- Implement proper state management and locking

**Key Principle**: Security and compliance by default

### Vault
When managing secrets with Vault:
- Apply principle of least privilege for all policies
- Use explicit deny rules for sensitive paths
- Implement proper authentication for context (AppRole, Kubernetes, etc.)
- Never log secret values
- Handle secrets securely in memory

**Key Principle**: Zero-trust secret access

### Vault Radar
When scanning for secrets and security risks:
- Treat all detected secrets as potentially compromised
- Prioritize by severity (Critical < 1hr, High < 24hr)
- Never rotate without also scrubbing git history
- Migrate to Vault/secrets manager, don't just rotate
- Implement layered prevention (pre-commit, CI/CD, real-time)

**Key Principle**: Complete remediation and prevention over quick fixes

### Consul
(Expanding - foundational examples provided)
- Service mesh configuration
- Service discovery patterns
- Health checking strategies

## Integration with Development Workflows

### During Code Review
```
@workspace Review this Terraform code against the hcl-best-practices guide
```

### During Planning
```
@workspace Summarize this Terraform plan using the summarize-plan prompt
```

### During Security Review
```
@workspace Run security checks following the security-scan-workflow
```

### During Refactoring
```
@workspace Apply the refactor-module skill to extract this into a reusable module
```

## Best Practices for AI Assistance

### 1. Provide Context
Always specify:
- Target environment (dev/staging/prod)
- Cloud provider and region
- Compliance requirements
- Organizational naming conventions

### 2. Iterative Refinement
- Start with requirements gathering
- Review generated code before applying
- Test in non-production first
- Apply security scanning

### 3. Security Validation
Before deployment:
- Run `terraform validate`
- Execute `tfsec`, `checkov`, or `terrascan`
- Review plan output carefully
- Verify compliance with policies

### 4. Documentation
Ensure AI-generated code includes:
- Clear variable descriptions
- Output documentation
- Usage examples
- Security considerations

## Customization

### Organization-Specific Instructions
Add your own instructions to `.github/INSTRUCTIONS/`:

```markdown
# company-terraform-standards.md
- Use prefix: company-env-resource
- Required tags: Environment, Owner, CostCenter, Project
- Encryption: Always enable for data at rest
- Networking: Private subnets by default
```

Then reference in prompts:
```
@workspace Following company-terraform-standards.md, create an S3 bucket
```

### Team-Specific Workflows
Create workflows for your team's processes:

```markdown
# .github/INSTRUCTIONS/team-deployment-process.md
1. Generate Terraform plan
2. Run security scan
3. Post plan to PR
4. Wait for 2 approvals
5. Apply during maintenance window
6. Verify deployment
7. Update documentation
```

## Tips for Effective Prompts

### Good Prompts ✅
- "Using the generate-hcl skill, create an RDS instance with encryption and backup retention"
- "Following Vault security best practices, create an AppRole policy for my web application"
- "Apply the security-scan-workflow to identify vulnerabilities in my Terraform code"

### Less Effective Prompts ❌
- "Make me a database" (lacks context)
- "Create a Terraform file" (no requirements specified)
- "Fix this" (missing specific issue description)

### Improvement Strategy
1. **Add Context**: Environment, provider, requirements
2. **Reference Skills**: Point to specific skill or workflow
3. **Specify Constraints**: Compliance, security, naming conventions
4. **Request Validation**: Ask for security scan, testing, documentation

## Contributing

When adding new instructions:

1. **Follow the Structure**:
   ```
   product/
   ├── skills/
   │   └── skill-name/
   │       ├── SKILL.md
   │       └── resources/
   ├── workflows/
   │   └── workflow-name.md
   └── prompts/
       └── prompt-name.md
   ```

2. **Use Templates**:
   - Skills: Include Overview, Capability Statement, Prerequisites, Steps, Examples
   - Workflows: Include Diagram, Prerequisites, Steps, Best Practices
   - Prompts: Include Purpose, Input, Output Format, Usage

3. **Provide Examples**:
   - Show both good and bad patterns
   - Include real-world scenarios
   - Demonstrate edge cases

4. **Test Instructions**:
   - Verify with actual AI agent
   - Check for completeness
   - Ensure security best practices

## Maintenance

### Regular Updates
- Review skills quarterly
- Update for new product versions
- Incorporate new best practices
- Add new compliance requirements

### Version Control
- Track changes in git
- Use semantic versioning for major updates
- Document changes in revision history
- Tag releases for stability

## Support and Feedback

### Getting Help
- Review existing skills and workflows first
- Check examples in resources/ directories
- Reference official HashiCorp documentation
- Consult with product-specific system prompts

### Reporting Issues
- Identify which skill/workflow has the issue
- Provide context (environment, versions)
- Suggest improvements if possible
- Test proposed changes

## License and Usage

This instruction library is designed for:
- Internal development teams
- AI-assisted development workflows
- Education and training
- Compliance and security automation

Remember: AI assistance is a tool to enhance, not replace, human expertise and judgment. Always review, understand, and validate AI-generated code before deployment.
