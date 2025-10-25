# Vault (Docker Compose) — Local Development

This repository contains a simple HashiCorp Vault server running in Docker Compose for local development and testing.

## What this repo contains

- `docker-compose.yml` — Compose file that runs the `hashicorp/vault` container and mounts local config/data directories/files.
- `vault-config.hcl` — HCL-format Vault configuration (mounted into the container at `/vault/config/vault-config.hcl`).
- `vault-config.json` — Original JSON config (kept for reference).
- `config/` — Directory mounted to `/vault/config` in the container (empty by default).
- `data/` — Directory used for file storage backend (persisted on the host).

Notes:
- The compose file was updated to remove the obsolete top-level `version:` key and to mount `vault-config.hcl` explicitly.
- The server is configured to use the file storage backend and an HTTP listener on port 8200 (TLS disabled). This is only for local development.

## Quick start (PowerShell)

1. Start the container in the background:

```powershell
docker compose up -d
```

2. Check Vault logs:

```powershell
docker compose logs --tail=200 vault
```

3. Initialize Vault (run inside the container). This generates unseal keys and the initial root token.

> Important: The init command prints sensitive unseal keys and a root token. Do NOT commit or share them. Store them securely.

```powershell
# Run init with VAULT_ADDR set to HTTP inside the container
docker exec -e VAULT_ADDR="http://127.0.0.1:8200" vault-vault-1 vault operator init -key-shares=1 -key-threshold=1
```

The command will print:
- Unseal Key(s)
- Initial Root Token

Save those values securely (or use a key management solution).

4. Unseal the Vault (use one of the printed unseal keys):

```powershell
# Example (replace <UNSEAL_KEY_1> with your real key):
docker exec -e VAULT_ADDR="http://127.0.0.1:8200" vault-vault-1 vault operator unseal <UNSEAL_KEY_1>
```

Repeat `vault operator unseal` until Vault reports `Sealed: false`.

5. Use the Vault CLI inside the container or from your host (set VAULT_ADDR and VAULT_TOKEN):

```powershell
# Using CLI inside the container (example to list mounts):
docker exec -e VAULT_ADDR="http://127.0.0.1:8200" -e VAULT_TOKEN="<INITIAL_ROOT_TOKEN>" vault-vault-1 vault status

# Or from host if you have the vault binary installed:
$env:VAULT_ADDR = 'http://127.0.0.1:8200'
$env:VAULT_TOKEN = '<INITIAL_ROOT_TOKEN>'
vault status
```

## Configuration details

- The active configuration is `vault-config.hcl` and is mounted to `/vault/config/vault-config.hcl` inside the container. It currently contains:
  - `storage "file" { path = "/vault/file" }`
  - `listener "tcp" { address = "0.0.0.0:8200", tls_disable = 1 }`
  - `ui = true`

- If you want to keep a JSON config, update `vault-config.json` to match the current Vault schema (use `storage` instead of the older `backend` key) and point the server `-config=` flag at the JSON file.

## Recommended improvements / production notes

- Do NOT disable TLS (tls_disable = 1) in production. Configure TLS certificates for the listener.
- Set `api_addr` in the config (or VAULT_API_ADDR env) so clients and other Vault nodes can correctly discover the server address. Example HCL:

```hcl
api_addr = "http://127.0.0.1:8200"
```

- Use an unsealing mechanism like Auto Unseal with a KMS (AWS KMS, Azure Key Vault, GCP KMS) for production.
- Use a durable storage backend appropriate for production (Consul, integrated storage auto-unlock, etc.) rather than plain file storage.
- Securely backup unseal keys and root token. Prefer generating root tokens on an as-needed basis and using approle/OIDC for apps.

## Cleaning up

To stop and remove containers and networks (data remains in `./data`):

```powershell
docker compose down
```

To remove the data directory (destructive):

```powershell
Remove-Item -Recurse -Force .\data
```

## Where to go next

- Add `api_addr` to `vault-config.hcl`.
- Replace `tls_disable = 1` with a proper TLS config for the listener.
- Add an initialization/unseal automation (if you want to auto-unseal for tests, consider `vault operator unseal` via a script or use auto-unseal in a secure way).
- Consider moving the HCL config into `config/` and simplifying mounts (e.g., mount the whole `config/` folder and place `vault-config.hcl` inside it).

If you want, I can:
- Move or convert the JSON config into a valid storage JSON and remove `vault-config.json` if unused.
- Add a small `init-and-unseal.ps1` script that initializes and unseals for local dev (prints warnings and stores keys in a file you control).
- Add `api_addr` to the HCL and update the compose file accordingly.

---

This README is intended for local development only. Follow Vault's official hardening and deployment guides for production setups: https://www.vaultproject.io/docs