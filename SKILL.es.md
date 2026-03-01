---
name: Vault Sentry
slug: vault-sentry
category: security
purpose: Secret management and access control for OpenClaw agents
tags:
  - secret
  - vault
  - security
  - access-control
  - credentials
  - encryption
version: 1.0.0
requires:
  - openclaw CLI
  - gpg (for encryption)
  - jq (for JSON parsing)
works_with:
  - smouj/RPGCLAW
  - smouj/flickclaw-saas
  - vps-ops
  - rpgclaw-ops
  - flickclaw-ops
---

# Vault Sentry (ES)

Secret management and access control layer for OpenClaw multi-agent system.

## Overview

Vault Sentry provides encrypted secret storage, rotation policies, and agent-level access controls for OpenClaw projects. All secrets are encrypted at rest using GPG and decrypted only in memory when explicitly requested by authorized agents.

## Architecture

```
Vault Sentry
├── ~/.openclaw/vault/           # Encrypted vault storage
│   ├── .secrets/               # GPG-encrypted secret files
│   ├── .keys/                  # Agent public keys
│   ├── .policies/              # Access control policies
│   └── vault.db                # Secret metadata index
├── /tmp/openclaw-vault/        # Temporary decrypted secrets (auto-cleaned)
└── /home/smouj/.openclaw/workspace/ops/
    └── SECRETS-REQUIRED.md     # Secret manifest reference
```

## Commands

### vault:init

Initialize Vault Sentry for a project.

```bash
openclaw vault:init --project <project-name> --agents <agent1,agent2>
```

Options:
- `--project` (required): Project name (rpgclaw, flickclaw, vps)
- `--agents` (required): Comma-separated list of authorized agents
- `--key-size`: RSA key size (default: 4096)
- `--encryption`: Encryption method (gpg, age) default: gpg

Example:
```bash
openclaw vault:init --project rpgclaw --agents rpgclaw-ops,vps-ops
```

### vault:add

Add a new secret to the vault.

```bash
openclaw vault:add --key <secret-name> --value <secret-value> --agents <authorized-agents> --tags <tag1,tag2>
```

Options:
- `--key` (required): Secret identifier (e.g., DATABASE_URL, AWS_KEY)
- `--value` (required): Secret value (or use `--stdin` for sensitive input)
- `--agents` (required): Agents authorized to access this secret
- `--tags`: Metadata tags for organization
- `--stdin`: Read secret value from stdin (recommended for sensitive data)
- `--rotate-after`: Auto-rotation policy (e.g., 30d, 90d)

Example - add database credentials:
```bash
echo "prod_db_pass_xyz123" | openclaw vault:add \
  --key DATABASE_PASSWORD \
  --value @stdin \
  --agents rpgclaw-ops,vps-ops \
  --tags database,production,critical \
  --rotate-after 30d
```

Example - add API key:
```bash
openclaw vault:add \
  --key STRIPE_API_KEY \
  --value sk_live_xxxxxxxxxxxxx \
  --agents flickclaw-ops \
  --tags payment,production \
  --rotate-after 90d
```

### vault:get

Retrieve a decrypted secret (only if authorized).

```bash
openclaw vault:get --key <secret-name> [--export] [--ttl <seconds>]
```

Options:
- `--key` (required): Secret identifier
- `--export`: Export to `.env` format
- `--ttl`: Time-to-live for temporary decryption (default: 300 seconds)
- `--env-prefix`: Prefix for exported variables (default: VAULT_)

Example:
```bash
# Get secret value
openclaw vault:get --key DATABASE_PASSWORD

# Export as environment variables
openclaw vault:get --key DATABASE_PASSWORD --export > .env

# Get with custom TTL
openclaw vault:get --key API_SECRET --ttl 60
```

### vault:list

List all secrets (metadata only, no values).

```bash
openclaw vault:list [--project <project>] [--tag <tag>] [--agent <agent>]
```

Options:
- `--project`: Filter by project
- `--tag`: Filter by tag
- `--agent`: Filter by authorized agent

Example:
```bash
# List all secrets for current project
openclaw vault:list

# List only database secrets
openclaw vault:list --tag database

# List secrets accessible by vps-ops
openclaw vault:list --agent vps-ops
```

### vault:rotate

Rotate a secret with automatic re-encryption.

```bash
openclaw vault:rotate --key <secret-name> --new-value <new-value>
```

Options:
- `--key` (required): Secret to rotate
- `--new-value` (required): New secret value (or use `--stdin`)
- `--stdin`: Read new value from stdin
- `--notify`: Send notification on rotation (requires notification config)

Example:
```bash
echo "new_database_password_456" | openclaw vault:rotate \
  --key DATABASE_PASSWORD \
  --new-value @stdin
```

### vault:revoke

Revoke agent access to a specific secret.

```bash
openclaw vault:revoke --key <secret-name> --agent <agent-name>
```

Example:
```bash
openclaw vault:revoke --key STRIPE_API_KEY --agent flickclaw-ops
```

### vault:audit

Generate access audit report.

```bash
openclaw vault:audit [--days <number>] [--format <format>] [--key <secret-name>]
```

Options:
- `--days`: Lookback period (default: 30)
- `--format`: Output format (json, csv, markdown)
- `--key`: Audit specific secret

Example:
```bash
# Full audit report
openclaw vault:audit

# Last 7 days in JSON
openclaw vault:audit --days 7 --format json

# Audit specific secret
openclaw vault:audit --key DATABASE_PASSWORD --format markdown
```

### vault:verify

Verify vault integrity and encryption status.

```bash
openclaw vault:verify [--fix] [--deep]
```

Options:
- `--fix`: Attempt automatic fixes
- `--deep`: Perform deep verification (includes decryption test)

Example:
```bash
# Quick verification
openclaw vault:verify

# Deep verification with auto-fix
openclaw vault:verify --deep --fix
```

### vault:cleanup

Securely remove expired temporary secrets.

```bash
openclaw vault:cleanup [--dry-run] [--older-than <days>]
```

Options:
- `--dry-run`: Show what would be cleaned without deleting
- `--older-than`: Clean secrets older than N days (default: 1)

## Use Cases

### Use Case 1: Initialize Vault for New Project

**Scenario**: Setting up FlickClaw SaaS with dedicated secret management.

```bash
# Initialize vault with all required agents
openclaw vault:init \
  --project flickclaw \
  --agents flickclaw-ops,flick-devops,vps-ops

# Add production secrets
openclaw vault:add --key DATABASE_URL \
  --value "postgresql://user:pass@host:5432/flickclaw" \
  --agents flickclaw-ops,vps-ops \
  --tags database,production

openclaw vault:add --key JWT_SECRET \
  --value @stdin \
  --agents flickclaw-ops \
  --tags auth,production,critical

# Verify setup
openclaw vault:verify --deep
```

### Use Case 2: Agent Secret Access During Task

**Scenario**: FLICKCLAW-OPS needs database credentials to run migrations.

```bash
# FLICKCLAW-OPS requests secret (auto-authorized based on policy)
openclaw vault:get --key DATABASE_URL --export > .env

# Run migration with secrets loaded
cd /tmp/flickclaw-work && npx prisma migrate deploy

# Secrets auto-expire after 5 minutes
```

### Use Case 3: Emergency Secret Rotation

**Scenario**: Security breach - rotate all production API keys immediately.

```bash
# Identify all critical secrets
openclaw vault:list --tag critical

# Rotate each critical secret
openclaw vault:rotate --key STRIPE_API_KEY --new-value @stdin
openclaw vault:rotate --key AWS_ACCESS_KEY --new-value @stdin
openclaw vault:rotate --key JWT_SECRET --new-value @stdin

# Generate audit report
openclaw vault:audit --format markdown > security-incident-audit.md

# Notify team
openclaw notify --channel security --message "Emergency rotation completed"
```

### Use Case 4: Cross-Project Secret Sharing

**Scenario**: FlickClaw needs access to shared RPGCLAW assets (with explicit approval).

```bash
# Add cross-project policy
openclaw vault:add --key SHARED_ASSETS_KEY \
  --value @stdin \
  --agents rpgclaw-ops,flickclaw-ops \
  --tags shared,assets

# Verify flickclaw-ops can access
openclaw vault:get --key SHARED_ASSETS_KEY
```

### Use Case 5: VPS Infrastructure Secrets

**Scenario**: Manage SSH keys, Docker registry credentials, and firewall rules.

```bash
# Add SSH key
openclaw vault:add --key SSH_PRIVATE_KEY \
  --value @stdin \
  --agents vps-ops \
  --tags ssh,infrastructure

# Add Docker registry
openclaw vault:add --key DOCKER_REGISTRY_TOKEN \
  --value @stdin \
  --agents vps-ops,flick-devops \
  --tags docker,infrastructure

# List infrastructure secrets
openclaw vault:list --tag infrastructure
```

### Use Case 6: Secret Access During CI/CD

**Scenario**: Automated deployment pipeline needs secret access.

```bash
# In CI pipeline script
- name: Load secrets
  run: |
    openclaw vault:get --key DEPLOY_API_KEY --export >> $GITHUB_ENV
    openclaw vault:get --key DATABASE_URL --export >> $GITHUB_ENV

- name: Deploy
  run: npm run deploy:production
```

## Troubleshooting

### Error: "Agent not authorized for this secret"

**Cause**: Current agent is not in the authorized list for the secret.

**Solution**:
```bash
# Check who has access
openclaw vault:list --key <secret-name>

# If you need access, add your agent
openclaw vault:add --key <secret-name> --agents +your-agent --append
```

### Error: "Vault not initialized"

**Cause**: Project vault has not been set up.

**Solution**:
```bash
openclaw vault:init --project <project-name> --agents <agent1,agent2>
```

### Error: "GPG decryption failed"

**Cause**: Agent's GPG key is missing or corrupted.

**Solution**:
```bash
# Re-import agent GPG key
gpg --import ~/.openclaw/vault/.keys/<agent>.gpg

# Or re-initialize
openclaw vault:init --project <project> --agents <agents> --force
```

### Warning: "Secret temp file not cleaned"

**Cause**: Previous operation was interrupted.

**Solution**:
```bash
# Manual cleanup
rm -rf /tmp/openclaw-vault/
openclaw vault:cleanup --older-than 0
```

### Error: "Secret not found"

**Cause**: Typo in secret name or secret was deleted.

**Solution**:
```bash
# List available secrets
openclaw vault:list

# Verify exact key name
openclaw vault:list --project <project> | grep <partial-name>
```

### Error: "Rotation policy violated"

**Cause**: Attempting to rotate before policy allows.

**Solution**:
```bash
# Force rotation (requires override)
openclaw vault:rotate --key <secret> --new-value @stdin --force
```

## Security Considerations

1. **Never log secrets**: Always use `--stdin` for sensitive values
2. **Auto-expiry**: Secrets in `/tmp/` are auto-deleted after TTL
3. **Agent isolation**: Each agent has dedicated GPG keypair
4. **Audit everything**: All access is logged with timestamp and agent ID
5. **Rotate regularly**: Use `--rotate-after` for automated rotation reminders

## Integration with OpenClaw Agents

### Automatic Secret Loading

When an authorized agent runs a task, secrets can be auto-loaded:

```bash
# In agent task execution
openclaw vault:load --tags production --env-prefix APP_
# Exports: APP_DATABASE_URL, APP_API_KEY, etc.
```

### Secret References in SECRETS-REQUIRED.md

Projects should maintain `ops/SECRETS-REQUIRED.md`:

```markdown
# Required Secrets - FlickClaw

## Database
- `DATABASE_URL` - PostgreSQL connection string
- `DATABASE_PASSWORD` - DB password (rotate: 30d)

## Authentication
- `JWT_SECRET` - JWT signing key (rotate: 90d, critical)
- `SESSION_SECRET` - Session encryption (rotate: 30d)

## External Services
- `STRIPE_API_KEY` - Payment processing
- `SENDGRID_API_KEY` - Email delivery
```

## Best Practices

1. Use descriptive secret names: `PRODUCTION_DB_PASSWORD` not `DB_PASS`
2. Tag secrets: `--tags production,database,critical`
3. Set rotation policies: `--rotate-after 30d`
4. Limit agent access: Only grant to agents that need the secret
5. Regular audits: Run `vault:audit` weekly
6. Test recovery: Verify you can restore secrets from backup

## Related Commands

- `openclaw agent:status` - Check agent authorization
- `openclaw vault:backup` - Backup encrypted vault
- `openclaw notify` - Send notifications on secret events
```