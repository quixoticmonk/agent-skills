# Workflow: New KV Engine Setup

## Overview
Step-by-step workflow for setting up a new Key-Value (KV) secrets engine in HashiCorp Vault with proper access controls, versioning, and organizational standards.

## Prerequisites
- Vault administrator access
- Understanding of organizational secret structure
- Access control requirements defined
- Naming conventions established

## Workflow Steps

### 1. Enable KV Secrets Engine

```bash
# Enable KV v2 at custom path
vault secrets enable -path=applications kv-v2

# Verify
vault secrets list

# Configure max versions (optional)
vault kv metadata put -max-versions=10 applications/
```

### 2. Configure Engine Settings

```bash
# Set default lease TTL
vault secrets tune -default-lease-ttl=1h applications/

# Set max lease TTL
vault secrets tune -max-lease-ttl=24h applications/

# Enable audit for this mount
vault secrets tune -audit-non-hmac-request-keys=data applications/
```

### 3. Create Directory Structure

```bash
# Create logical structure
vault kv put applications/shared/common placeholder=true
vault kv put applications/prod/placeholder placeholder=true
vault kv put applications/staging/placeholder placeholder=true
vault kv put applications/dev/placeholder placeholder=true

# Remove placeholders
vault kv metadata delete applications/shared/common
vault kv metadata delete applications/prod/placeholder
vault kv metadata delete applications/staging/placeholder
vault kv metadata delete applications/dev/placeholder
```

### 4. Create Access Policies

```hcl
# policies/applications-admin.hcl
path "applications/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "applications/metadata/*" {
  capabilities = ["read", "list", "delete"]
}

# policies/applications-dev-read.hcl
path "applications/data/dev/*" {
  capabilities = ["read", "list"]
}

path "applications/data/staging/*" {
  capabilities = ["read", "list"]
}

path "applications/data/prod/*" {
  capabilities = ["deny"]
}
```

```bash
# Write policies
vault policy write applications-admin policies/applications-admin.hcl
vault policy write applications-dev-read policies/applications-dev-read.hcl
```

### 5. Test Access

```bash
# Create test token
TEST_TOKEN=$(vault token create -policy=applications-dev-read -format=json | jq -r '.auth.client_token')

# Test read access
VAULT_TOKEN=$TEST_TOKEN vault kv get applications/dev/test

# Test denied access
VAULT_TOKEN=$TEST_TOKEN vault kv get applications/prod/test  # Should fail

# Cleanup
vault token revoke $TEST_TOKEN
```

### 6. Documentation

```markdown
# Applications KV Engine

**Path**: `applications/`
**Type**: KV v2
**Purpose**: Application configuration secrets

## Structure
- `applications/dev/*` - Development environment
- `applications/staging/*` - Staging environment
- `applications/prod/*` - Production environment
- `applications/shared/*` - Shared configurations

## Access Policies
- `applications-admin` - Full access
- `applications-dev-read` - Read dev/staging only
- `applications-prod-read` - Read production only

## Retention
- Maximum 10 versions retained
- Versions auto-deleted after 30 days
```

## Success Criteria
- [ ] Engine enabled and configured
- [ ] Directory structure created
- [ ] Access policies defined and tested
- [ ] Documentation complete
- [ ] Team notified of new engine

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-11-07 | Initial workflow definition |
