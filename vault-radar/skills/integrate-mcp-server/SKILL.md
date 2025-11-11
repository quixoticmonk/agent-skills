# Skill: Integrate Vault Radar MCP Server

## Overview

This skill enables AI agents to integrate with the HCP Vault Radar Model Context Protocol (MCP) server to query and analyze secrets, PII, and non-inclusive language detected in codebases. The MCP server provides a standardized interface for AI models to interact with Vault Radar's scanning capabilities and retrieve actionable security insights.

## Capability Statement

**As an AI agent, I can configure and integrate with the Vault Radar MCP server to:**
- Deploy and configure the MCP server container locally
- Query data sources connected to Vault Radar
- Retrieve detected secrets, PII, and non-inclusive language events
- List secret types found across repositories
- Analyze security posture and recommend remediation steps

## Prerequisites

### Environment Requirements
- Docker installed and running locally
- HCP Vault Radar project created and configured
- Service principal credentials for HCP authentication
- Network access to HCP APIs (`api.cloud.hashicorp.com`, `auth.idp.hashicorp.com`)

### Required Information
- **HCP Organization ID**: Your HashiCorp Cloud Platform organization identifier
- **HCP Project ID**: The Vault Radar project ID
- **Client ID**: Service principal client ID
- **Client Secret**: Service principal client secret (store securely)

### Knowledge Prerequisites
- Understanding of MCP (Model Context Protocol) architecture
- Familiarity with Docker container deployment
- Basic knowledge of secrets management principles
- Understanding of severity levels and risk categories

## Input Parameters

When deploying the Vault Radar MCP server, collect:

```yaml
hcp_organization_id: string       # HCP org ID (required)
hcp_project_id: string            # Vault Radar project ID (required)
client_id: string                 # Service principal client ID (required)
client_secret: string             # Service principal client secret (required, sensitive)
container_name: string            # Docker container name (default: "vault-radar-mcp")
port_mapping: string              # Port mapping (default: "3000:3000")
```

## Execution Steps

### Step 1: Validate Prerequisites

```bash
# Verify Docker is running
docker --version
docker ps

# Verify network connectivity to HCP
curl -I https://api.cloud.hashicorp.com/health

# Verify credentials are available
echo $HCP_CLIENT_ID
echo $HCP_CLIENT_SECRET  # Should be set but not displayed
```

### Step 2: Deploy MCP Server Container

```bash
# Pull the latest Vault Radar MCP server image
docker pull hashicorp/vault-radar-mcp:latest

# Run the container with HCP credentials
docker run -d \
  --name vault-radar-mcp \
  -p 3000:3000 \
  -e HCP_ORGANIZATION_ID="${HCP_ORG_ID}" \
  -e HCP_PROJECT_ID="${HCP_PROJECT_ID}" \
  -e HCP_CLIENT_ID="${HCP_CLIENT_ID}" \
  -e HCP_CLIENT_SECRET="${HCP_CLIENT_SECRET}" \
  hashicorp/vault-radar-mcp:latest

# Verify container is running
docker logs vault-radar-mcp
```

### Step 3: Configure AI Client Integration

**For Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "vault-radar": {
      "command": "docker",
      "args": [
        "exec",
        "-i",
        "vault-radar-mcp",
        "node",
        "/app/build/index.js"
      ],
      "env": {
        "HCP_ORGANIZATION_ID": "your-org-id",
        "HCP_PROJECT_ID": "your-project-id",
        "HCP_CLIENT_ID": "your-client-id",
        "HCP_CLIENT_SECRET": "your-client-secret"
      }
    }
  }
}
```

**For VS Code with Copilot/MCP Extension**:

```json
{
  "mcp.servers": {
    "vault-radar": {
      "endpoint": "http://localhost:3000",
      "authentication": {
        "type": "bearer",
        "token": "${env:HCP_CLIENT_SECRET}"
      }
    }
  }
}
```

### Step 4: Test MCP Server Tools

Query available tools from your AI client:

```
What tools are available from the vault-radar MCP server?
```

Expected tools:
- `query_vault_radar_data_sources`: List connected repositories and sources
- `query_vault_radar_resources`: List all resources being monitored
- `query_vault_radar_events`: Retrieve detected secrets/PII/NIL events
- `list_vault_radar_secret_types`: List types of secrets detected

### Step 5: Query Data Sources

```
Using vault-radar MCP server, list all data sources in my project.
```

Expected output structure:
```json
{
  "data_sources": [
    {
      "id": "ds-abc123",
      "name": "my-app-repo",
      "type": "github",
      "url": "https://github.com/org/my-app-repo",
      "status": "active",
      "last_scan": "2025-11-10T10:30:00Z"
    }
  ]
}
```

### Step 6: Query Detected Events

```
Using vault-radar MCP server, show me critical and high severity secrets 
found in the last 7 days.
```

Response includes:
```json
{
  "events": [
    {
      "id": "evt-xyz789",
      "severity": "critical",
      "type": "aws_access_key_id",
      "status": "active",
      "data_source": "my-app-repo",
      "file_path": "src/config/aws.js",
      "line_number": 42,
      "detected_at": "2025-11-09T14:22:00Z",
      "risk_category": "secret"
    }
  ]
}
```

### Step 7: Analyze and Recommend Remediation

Once events are retrieved, analyze:

1. **Severity Assessment**:
   - Critical: Active secrets in latest version
   - High: Active secrets in older versions
   - Medium: Default for unclassified
   - Low: PII or test file secrets
   - Info: NIL or inactive secrets

2. **Categorization**:
   - Secrets: Rotate immediately
   - PII: Remove and scrub history
   - NIL: Replace with inclusive alternatives

3. **Actionable Steps**:
   - Generate rotation scripts
   - Create git history scrubbing commands
   - Suggest preventive controls (pre-commit hooks)

## Output Format

### Summary Report

```markdown
# Vault Radar Security Analysis

**Project**: {project_name}
**Scan Date**: {timestamp}
**Data Sources**: {count}

## Risk Summary

| Severity | Secrets | PII | NIL | Total |
|----------|---------|-----|-----|-------|
| Critical | 3       | 0   | 0   | 3     |
| High     | 7       | 2   | 0   | 9     |
| Medium   | 15      | 5   | 3   | 23    |
| Low      | 8       | 12  | 7   | 27    |

## Critical Issues Requiring Immediate Action

1. **AWS Access Key** (evt-abc123)
   - Location: `src/config/aws.js:42`
   - Status: Active
   - Action: Rotate immediately using AWS IAM
   - Command: `aws iam delete-access-key --access-key-id AKIA...`

2. **Database Password** (evt-def456)
   - Location: `config/database.yml:18`
   - Status: Active
   - Action: Move to Vault secrets management
   - Migration: See vault/workflows/migrate-to-vault.md

## Remediation Workflow

- [ ] Rotate critical secrets (3 items)
- [ ] Remove PII from codebase (2 items)
- [ ] Scrub git history for sensitive data
- [ ] Implement pre-commit scanning
- [ ] Configure CI/CD integration
```

### Detailed Event List

```json
{
  "analysis_timestamp": "2025-11-10T12:00:00Z",
  "project_id": "prj-vault-radar-123",
  "summary": {
    "total_events": 62,
    "by_severity": {
      "critical": 3,
      "high": 9,
      "medium": 23,
      "low": 27,
      "info": 0
    },
    "by_category": {
      "secrets": 33,
      "pii": 19,
      "nil": 10
    }
  },
  "critical_events": [
    {
      "event_id": "evt-abc123",
      "severity": "critical",
      "category": "secret",
      "type": "aws_access_key_id",
      "status": "active",
      "data_source": {
        "name": "my-app-repo",
        "type": "github",
        "url": "https://github.com/org/my-app-repo"
      },
      "location": {
        "file_path": "src/config/aws.js",
        "line_number": 42,
        "commit_hash": "a1b2c3d4",
        "branch": "main",
        "is_latest": true
      },
      "detected_at": "2025-11-09T14:22:00Z",
      "remediation": {
        "priority": "immediate",
        "steps": [
          "Rotate AWS access key immediately",
          "Update application configuration",
          "Audit access logs for unauthorized usage",
          "Implement secrets management (Vault/AWS Secrets Manager)"
        ],
        "estimated_time": "15 minutes"
      }
    }
  ],
  "recommendations": [
    "Enable Vault Radar GitHub app for PR scanning",
    "Configure pre-commit hooks with vault-radar CLI",
    "Implement secrets baseline scan in CI/CD",
    "Establish secrets rotation policy",
    "Create incident response runbook"
  ]
}
```

## Best Practices

### Security

1. **Credential Management**:
   - ✅ Store HCP credentials in environment variables or secure vault
   - ✅ Use service principals with least-privilege permissions
   - ✅ Rotate service principal credentials regularly (90 days)
   - ❌ Never commit HCP credentials to version control
   - ❌ Never log service principal secrets

2. **Network Security**:
   - ✅ Allowlist HCP IP addresses if using firewall rules
   - ✅ Use TLS for all HCP API communication
   - ✅ Monitor MCP server logs for anomalous access
   - ❌ Don't expose MCP server port publicly

3. **Container Security**:
   - ✅ Pull verified HashiCorp images only
   - ✅ Keep MCP server container updated
   - ✅ Run container with minimal privileges
   - ✅ Use Docker secrets or environment files for credentials

### Performance

1. **Query Optimization**:
   - Use filters to limit event queries (severity, date range)
   - Paginate large result sets
   - Cache frequently accessed data sources list
   - Schedule bulk analysis during off-peak hours

2. **Container Resources**:
   ```bash
   # Set resource limits for production
   docker run -d \
     --name vault-radar-mcp \
     --memory="512m" \
     --cpus="0.5" \
     -p 3000:3000 \
     -e HCP_ORGANIZATION_ID="${HCP_ORG_ID}" \
     -e HCP_PROJECT_ID="${HCP_PROJECT_ID}" \
     -e HCP_CLIENT_ID="${HCP_CLIENT_ID}" \
     -e HCP_CLIENT_SECRET="${HCP_CLIENT_SECRET}" \
     hashicorp/vault-radar-mcp:latest
   ```

### Operational

1. **Monitoring**:
   ```bash
   # Check container health
   docker inspect vault-radar-mcp --format='{{.State.Health.Status}}'
   
   # View logs
   docker logs -f vault-radar-mcp
   
   # Monitor resource usage
   docker stats vault-radar-mcp
   ```

2. **Troubleshooting**:
   - **Authentication failures**: Verify service principal has Vault Radar access
   - **Network errors**: Check firewall rules for HCP FQDN allowlisting
   - **Empty results**: Confirm project has completed initial scan
   - **Container crashes**: Check memory limits and logs

3. **Updates**:
   ```bash
   # Update to latest version
   docker pull hashicorp/vault-radar-mcp:latest
   docker stop vault-radar-mcp
   docker rm vault-radar-mcp
   # Re-run with updated image
   ```

## Common Pitfalls

### ❌ Authentication Issues

**Problem**: "Unauthorized" errors when querying MCP server

**Causes**:
- Service principal lacks Vault Radar project access
- Credentials expired or revoked
- Wrong project ID or organization ID

**Solutions**:
```bash
# Verify credentials
vault-radar-cli auth verify \
  --client-id="${HCP_CLIENT_ID}" \
  --client-secret="${HCP_CLIENT_SECRET}"

# Check service principal permissions in HCP console
# Navigate to: Access Control (IAM) > Service Principals
# Ensure principal has "Vault Radar Viewer" or "Vault Radar Admin" role
```

### ❌ No Events Returned

**Problem**: MCP queries return empty results despite visible events in UI

**Causes**:
- Initial scan not completed
- Querying wrong project
- Overly restrictive filters

**Solutions**:
```bash
# Check scan status
curl -H "Authorization: Bearer ${HCP_TOKEN}" \
  https://api.cloud.hashicorp.com/vault-radar/2023-05-01/organizations/${ORG_ID}/projects/${PROJECT_ID}/data-sources

# Query without filters first
# Then gradually add filters to narrow results
```

### ❌ Network Connectivity

**Problem**: Cannot reach HCP APIs from MCP server

**Causes**:
- Firewall blocking outbound connections
- DNS resolution failures
- Corporate proxy not configured

**Solutions**:
```bash
# Test connectivity from container
docker exec vault-radar-mcp curl -I https://api.cloud.hashicorp.com/health

# Configure proxy if needed
docker run -d \
  --name vault-radar-mcp \
  -e HTTP_PROXY="http://proxy.corp:8080" \
  -e HTTPS_PROXY="http://proxy.corp:8080" \
  -e NO_PROXY="localhost,127.0.0.1" \
  ...
```

### ❌ Performance Degradation

**Problem**: Slow query response times

**Causes**:
- Large project with thousands of events
- No pagination on queries
- Container resource constraints

**Solutions**:
```typescript
// Use pagination for large datasets
const events = await mcpClient.queryEvents({
  severity: ["critical", "high"],
  limit: 50,
  offset: 0,
  date_range: {
    start: "2025-11-01",
    end: "2025-11-10"
  }
});

// Filter at query time, not in application
// Use specific severity levels and date ranges
```

## Related Skills

- `vault-radar/skills/scan-for-secrets/` - Using CLI for offline scanning
- `vault/skills/generate-policy/` - Creating Vault policies for secrets migration
- `terraform/workflows/security-scan-workflow.md` - Integrating into CI/CD

## Related Workflows

- `vault-radar/workflows/triage-and-remediate.md` - Handling detected secrets
- `vault-radar/workflows/git-history-scrubbing.md` - Removing secrets from history

## Resources

### Official Documentation
- [Vault Radar MCP Server Overview](https://developer.hashicorp.com/hcp/docs/vault-radar/mcp-server/overview)
- [Vault Radar Documentation](https://developer.hashicorp.com/hcp/docs/vault-radar)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)

### Example Queries

**Query by Severity**:
```
Show me all critical secrets detected in the last 30 days
```

**Query by Type**:
```
List all AWS access keys and their active status
```

**Query by Data Source**:
```
What secrets were found in the 'production-app' repository?
```

**Aggregate Analysis**:
```
Provide a summary of PII findings across all data sources
```

### HCP IP Allowlist

If scanning internal repositories, allowlist these IPs:

```
# Primary
3.213.172.245
44.215.93.123
34.226.175.235

# Failover
34.208.33.38
44.227.97.170
44.238.204.12
```

### Required FQDNs for Outbound Traffic

```
api.cloud.hashicorp.com
auth.idp.hashicorp.com
```

## Validation

### Test Checklist

- [ ] MCP server container running and healthy
- [ ] Successfully authenticate with HCP
- [ ] Query data sources returns expected repositories
- [ ] Query events returns detected secrets/PII/NIL
- [ ] Severity filtering works correctly
- [ ] Date range filtering works correctly
- [ ] Secret type listing is accurate
- [ ] Response times are acceptable (<2s for typical queries)
- [ ] Error handling works for invalid queries
- [ ] Logs show no authentication errors

### Integration Test

```typescript
// Example test for MCP server integration
import { MCPClient } from '@modelcontextprotocol/sdk';

async function testVaultRadarMCP() {
  const client = new MCPClient({
    serverUrl: 'http://localhost:3000'
  });
  
  // Test 1: Query data sources
  const sources = await client.call('query_vault_radar_data_sources', {});
  console.assert(sources.data_sources.length > 0, 'Should have data sources');
  
  // Test 2: Query critical events
  const events = await client.call('query_vault_radar_events', {
    severity: ['critical']
  });
  console.log(`Found ${events.events.length} critical events`);
  
  // Test 3: List secret types
  const types = await client.call('list_vault_radar_secret_types', {});
  console.assert(types.secret_types.includes('aws_access_key_id'), 
    'Should detect AWS keys');
    
  console.log('All MCP integration tests passed ✅');
}
```

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-11-10 | Initial skill creation based on Vault Radar MCP server beta release |
