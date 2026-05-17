# HashiCorp Vault Server — Configuration & Security Reference

## Overview

Self-hosted HashiCorp Vault secret management server deployed in a homelab environment.
Configured with a focus on least-privilege access, network isolation, TLS enforcement,
and operational security practices.

| Property | Value |
|---|---|
| **Vault Version** | 2.0.0 |
| **Deployment Model** | Single-node, systemd-managed |
| **Storage Backend** | Raft (integrated) |
| **TLS** | Let's Encrypt — automated renewal |
| **Accessibility** | LAN-only, no public internet exposure |
| **Auto-Unseal** | TPM2-backed systemd service |

---

## Infrastructure & Deployment

### Platform
- **OS:** Ubuntu Linux
- **Process management:** systemd
- **Vault binary:** Installed via official HashiCorp package repository

### Vault Configuration

```hcl
ui = true

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/etc/letsencrypt/live/<vault-hostname>/fullchain.pem"
  tls_key_file  = "/etc/letsencrypt/live/<vault-hostname>/privkey.pem"
}

api_addr     = "https://<vault-hostname>:8200"
cluster_addr = "https://<vault-hostname>:8201"
```

### High Availability
Raft integrated storage is used as the backend, eliminating the need for an external
storage dependency such as Consul. HA is enabled and the Raft cluster is ready to
accept additional nodes without configuration changes.

---

## TLS Configuration

### Certificate
- **Provider:** Let's Encrypt (ACME v2)
- **Validation method:** Cloudflare DNS API (dns-01 challenge)
- **Renewal:** Automated via Certbot

### Cert Renewal Deploy Hook

A deploy hook runs automatically after each successful cert renewal. It restarts Vault,
waits for it to come up, and verifies it is healthy before exiting. Failures are written
to the systemd journal for visibility.

```bash
#!/bin/bash
systemctl restart vault

sleep 3

if ! VAULT_ADDR="https://127.0.0.1:8200" vault status > /dev/null 2>&1; then
  echo "ERROR: Vault failed to restart after cert renewal" \
    | systemd-cat -t vault-cert-renewal -p err
  exit 1
fi

echo "Vault restarted successfully after cert renewal" \
  | systemd-cat -t vault-cert-renewal -p info
```

---

## Seal Configuration

### Shamir Unseal
Vault's master key is split using Shamir's Secret Sharing:

| Property | Value |
|---|---|
| **Total key shares** | 5 |
| **Unseal threshold** | 3 of 5 |

### Auto-Unseal — TPM2
A dedicated systemd oneshot service unseals Vault automatically on boot using keys
protected by the host TPM2 chip. This eliminates the need for manual unseal key entry
after restarts while keeping unseal material off the filesystem.

```ini
[Unit]
Description=Vault Auto-Unseal via TPM2
After=vault.service network-online.target
Requires=vault.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/vault-unseal.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

---

## Authentication

### Auth Method: Userpass

Username/password is the primary auth method. Accounts are separated by intent rather
than by person — each account carries only the permissions required for its specific purpose.

| Account | Intent | Policies Assigned |
|---|---|---|
| Personal Account | Day-to-day personal project secret access | `operator`, `personal-rw` |
| Professional Account | Professional/shared project secret access | `homelab-api-keys-rw`, `professional-rw` |
| Admin Account | Vault administration only (users, policies) | `vault-admin` |

**Design decision:** The admin account is only used when making changes to Vault itself
(creating users, writing policies, modifying auth). All routine secret access uses a
least-privileged account scoped to the required paths. This limits the blast radius of
a compromised session token.

---

## Authorization — Policies

### `vault-admin`
Full administrative access to manage Vault configuration, auth methods, policies, and users.
Assigned exclusively to the admin account. Not used for secret access.

---

### `operator`
Standard operational access for the personal account. Grants full CRUD on personal and
homelab secret paths, and the ability to list the secrets engine root for UI navigation.

```hcl
# Navigate the secrets engine root in the UI
path "secret/metadata/" {
  capabilities = ["list"]
}

# Full access to personal secrets namespace
path "secret/data/personal/*" {
  capabilities = ["create", "read", "update", "delete", "list", "patch"]
}

# Full access to homelab secrets namespace
path "secret/data/homelab/*" {
  capabilities = ["create", "read", "update", "delete", "list", "patch"]
}

# Manage all secret metadata (list, read, delete)
path "secret/metadata/*" {
  capabilities = ["read", "list", "delete"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}

path "auth/token/revoke-self" {
  capabilities = ["update"]
}
```

---

### `personal-rw`
Scoped to the personal secrets namespace. Paired with `operator` on the personal account.

```hcl
path "secret/data/personal/*" {
  capabilities = ["create", "read", "update", "delete", "list", "patch"]
}

path "secret/metadata/personal/*" {
  capabilities = ["read", "list", "delete"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}
```

---

### `professional-rw`
Scoped to the professional secrets namespace. Assigned to the professional account for
storing and managing work-related secrets independently from personal projects.

```hcl
path "secret/data/professional/*" {
  capabilities = ["create", "read", "update", "delete", "list", "patch"]
}

path "secret/metadata/professional/*" {
  capabilities = ["read", "list", "delete"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

path "auth/token/renew-self" {
  capabilities = ["update"]
}

path "auth/token/revoke-self" {
  capabilities = ["update"]
}
```

---

### `homelab-api-keys-rw`
Tightly scoped access for the professional account. Grants navigation and read/write
access only to the shared API keys path — **not** the broader homelab namespace. The
professional account cannot access `homelab/services/` or `homelab/systems/` even though
those paths are visible in the folder listing.

This demonstrates the principle of least privilege: grant the minimum path access required,
even when adjacent paths exist under the same parent.

```hcl
# Navigate the secrets engine root in the UI
path "secret/metadata/" {
  capabilities = ["list"]
}

# Navigate into the homelab folder (folder name visible, contents not accessible)
path "secret/metadata/homelab/" {
  capabilities = ["list"]
}

# Navigate into the api-keys folder
path "secret/metadata/homelab/api-keys/" {
  capabilities = ["list"]
}

# Full CRUD on api-keys secrets only
path "secret/data/homelab/api-keys/*" {
  capabilities = ["create", "read", "update", "delete", "list", "patch"]
}

path "secret/metadata/homelab/api-keys/*" {
  capabilities = ["read", "list", "delete"]
}
```

---

## Secrets Organisation

### Engine
KV v2 secret engine mounted at `secret/`. KV v2 provides secret versioning and
soft-delete with metadata tracking.

### Path Structure

```
secret/
├── personal/           # Personal project secrets    — personal account only
├── professional/       # Professional project secrets — professional account only
└── homelab/
    ├── api-keys/       # Shared API keys             — both accounts (full CRUD)
    ├── services/       # Homelab service credentials — personal account only
    └── systems/        # Homelab system credentials  — personal account only
```

**Design decision:** Separating personal and professional namespaces at the path level
means access policies are clean and auditable. Adding a future collaborator to the
professional account requires no changes to personal secret paths or the personal account.

---

## Network Security

### Firewall — UFW

**Default policy:** deny all incoming, allow all outgoing.

| Port | Protocol | Allowed From | Purpose |
|---|---|---|---|
| 22/tcp | TCP | 192.168.20.0/24 | SSH administration |
| 8200/tcp | TCP | 192.168.20.0/24 | Vault API & UI |
| 8200/tcp | TCP | 192.168.1.0/24 | Vault API & UI |
| 8201/tcp | TCP | 192.168.20.0/24 | Vault Raft cluster port |

Vault is not exposed to the public internet. Port 8200 is accessible only from trusted
private LAN subnets. There are no public-facing rules.

### DNS — Split Resolution
The Vault hostname resolves to a LAN IP via local DNS override, ensuring LAN clients
connect directly to the internal address rather than routing through a public IP. This
is necessary because the TLS certificate is issued for the hostname, not the raw IP
address.

### Memory Protection
The Vault process runs with `IPC_LOCK` capability, preventing the operating system from
swapping Vault's memory pages to disk. This ensures secrets held in memory cannot be
recovered from a swap file or hibernation image.

---

## Operational Security Practices

### No tokens persisted to disk
`~/.vault-token` is removed after each session. Tokens are not stored in shell profiles,
environment files, or configuration files. Each session authenticates explicitly and the
resulting token lives only in memory for the duration of that session.

### Admin / operator separation
Administrative operations (policy changes, user creation, auth method configuration) use
a dedicated high-privilege admin account. All routine secret access uses a least-privilege
account. This limits the exposure of a compromised operator token — it cannot modify
policies or create new users.

### Secure password generation
User account passwords are generated using `openssl rand -base64 48` (288 bits of
entropy), written to a temporary file under `/tmp`, passed to Vault via file reference
(never via shell argument to avoid process table exposure), then immediately wiped using
`shred -u`. The password is never stored on the server.

```bash
openssl rand -base64 48 | tr -d '\n' | tee /tmp/vault-pass.txt
vault write auth/userpass/users/<username> \
  password=@/tmp/vault-pass.txt \
  policies="<policy-list>"
sudo shred -u /tmp/vault-pass.txt
```

### TLS enforced — no plaintext fallback
HTTP is disabled at the listener level. All communication with Vault requires HTTPS.
There is no fallback to unencrypted transport.

### Automated cert renewal with health verification
Certificate renewal triggers a deploy hook that restarts Vault and actively verifies it
returns to a healthy state. If Vault fails to come back up, an error is written to the
systemd journal. This prevents a silent outage after an automated cert renewal.

### Bash history hygiene
Sensitive commands (passwords, tokens, API keys) are never entered directly at the shell
prompt where possible. Where shell history exposure was identified, history was cleared
and credentials rotated.
