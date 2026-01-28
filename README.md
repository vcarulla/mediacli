# Mediacli

Media server stack with Docker Compose and HashiCorp Vault for secrets management.

## Services

| Service | Description | Port |
|---------|-------------|------|
| **PVR** |||
| Sonarr | TV shows management | 8989 |
| Radarr | Movies management | 7878 |
| Bazarr | Subtitles management | 6767 |
| **Indexers** |||
| Prowlarr | Indexer manager | 9696 |
| Jackett | Indexer proxy | 9117 |
| Flaresolverr | Cloudflare bypass | 8191 |
| **Downloader** |||
| qBittorrent | Torrent client | 8080 |
| **Media Servers** |||
| Emby | Media server | 8096 |
| Plex | Media server (profile) | 32400 |
| **Requests** |||
| Jellyseerr | Request management | 5055 |
| Ombi | Request management (profile) | 3579 |
| Requestrr | Discord bot (profile) | 4545 |
| **Utilities** |||
| Dispatcharr | Notifications | 8282 |
| Recyclarr | Sync tool (profile) | - |

## Prerequisites

- Docker and Docker Compose
- HashiCorp Vault server (can be remote)
- AppRole auth method configured in Vault
- Docker network created: `docker network create media_net`

## Quick Start

### 1. Clone and configure

```bash
git clone https://github.com/your-user/mediacli.git
cd mediacli

# Copy example configurations
cp .env.example .env
cp public/vault-agent/agent.hcl.example vault-agent/agent.hcl
cp public/vault-agent/docker-compose.yml.example vault-agent/docker-compose.yml
cp -r public/vault-agent/templates vault-agent/
```

### 2. Edit configuration files

Edit `.env` with your values:
```bash
PUID=1000
PGID=1000
TZ=America/New_York
HOSTNAME=media.example.com
CONFIG_BASE_PATH=/path/to/config
DATA_PATH=/path/to/data
DOWNLOADS_PATH=/path/to/downloads
VAULT_ADDR=https://vault.example.com
```

Edit `vault-agent/agent.hcl`:
```hcl
vault {
  address = "https://vault.example.com"  # Your Vault address
}
```

### 3. Configure Vault secrets

Create the following secrets in Vault:
```bash
# Environment variables
vault kv put secret/mediacli/env \
  PUID=1000 PGID=1000 UMASK=002 \
  TZ="America/New_York" \
  HOSTNAME="media.example.com" \
  CONFIG_BASE_PATH="/path/to/config" \
  DATA_PATH="/path/to/data"

# API keys (get these from each service after first run)
vault kv put secret/mediacli/sonarr api_key="your-sonarr-api-key"
vault kv put secret/mediacli/radarr api_key="your-radarr-api-key"
vault kv put secret/mediacli/prowlarr api_key="your-prowlarr-api-key"
vault kv put secret/mediacli/bazarr api_key="your-bazarr-api-key"
vault kv put secret/mediacli/jellyseerr api_key="your-jellyseerr-api-key"
vault kv put secret/mediacli/qbittorrent username="admin" password="your-password"
```

### 4. Create AppRole in Vault

```bash
# Enable AppRole
vault auth enable approle

# Create policy
vault policy write mediacli-policy - <<POLICY
path "secret/data/mediacli/*" {
  capabilities = ["read"]
}
POLICY

# Create role
vault write auth/approle/role/mediacli-agent \
  token_policies="mediacli-policy" \
  token_ttl=1h \
  token_max_ttl=4h
```

### 5. Bootstrap and start

```bash
# First time: manually start vault-agent with credentials
export VAULT_ROLE_ID=$(vault read -field=role_id auth/approle/role/mediacli-agent/role-id)
export VAULT_SECRET_ID=$(vault write -f -field=secret_id auth/approle/role/mediacli-agent/secret-id)

# Write credentials to tmpfs
sudo mkdir -p /run/mediacli-vault
echo "$VAULT_ROLE_ID" > /run/mediacli-vault/role-id
echo "$VAULT_SECRET_ID" > /run/mediacli-vault/secret-id
chmod 644 /run/mediacli-vault/*

# Start vault-agent
cd vault-agent
docker compose --env-file ../.env up -d
cd ..

# Wait for secrets, then start services
docker compose up -d
```

## After Reboot

Credentials in `/run/mediacli-vault/` are lost on reboot (tmpfs). Run the bootstrap script:

```bash
# If you have a bootstrap script in private/scripts/
./private/scripts/vault-bootstrap.sh

# Or manually repeat step 5
```

## Profiles

```bash
# Default services
docker compose up -d

# Include Plex
docker compose --profile plex up -d

# Include sync tools (recyclarr)
docker compose --profile sync up -d

# Include alternative request managers (ombi, requestrr)
docker compose --profile requests-alt up -d
```

## Structure

```
mediacli/
├── docker-compose.yml    # Main orchestrator
├── *.yml                 # Individual service definitions
├── .env                  # Configuration (gitignored)
├── .env.example          # Configuration template
├── vault-agent/          # Vault Agent (gitignored, use public/ examples)
├── public/               # Example configurations
└── private/              # Private docs and scripts (gitignored)
```

## Security Model

- **AppRole credentials**: Stored in `/run/mediacli-vault/` (tmpfs, lost on reboot)
- **Secrets (API keys)**: Docker tmpfs volume (RAM only)
- **Configuration**: `.env` file (gitignored)

No secrets are ever written to persistent storage.
