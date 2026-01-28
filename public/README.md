# Public Examples

This folder contains example configurations with placeholder values.

## Setup

1. Copy example files to their target locations:

```bash
# Vault Agent
cp vault-agent/agent.hcl.example ../vault-agent/agent.hcl
cp vault-agent/docker-compose.yml.example ../vault-agent/docker-compose.yml
cp -r vault-agent/templates ../vault-agent/

# Environment
cp ../.env.example ../.env
```

2. Edit the copied files and replace placeholders:

| Placeholder | Description |
|-------------|-------------|
| `vault.example.com` | Your Vault server address |
| `media_net` | Your Docker network name |
| `/path/to/...` | Your actual paths |

## Files

| File | Target | Description |
|------|--------|-------------|
| `vault-agent/agent.hcl.example` | `vault-agent/agent.hcl` | Vault Agent configuration |
| `vault-agent/docker-compose.yml.example` | `vault-agent/docker-compose.yml` | Vault Agent compose |
| `vault-agent/templates/mediacli.env.tpl` | `vault-agent/templates/` | Secrets template |

## Architecture

```
Vault (remote) --> Vault Agent --> Docker tmpfs volume --> Services
                       ^
                AppRole credentials
                (tmpfs, lost on reboot)
```

Secrets exist only in RAM (Docker tmpfs volume), never on disk.
