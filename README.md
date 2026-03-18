# rhis-builder-quay

Deploy [Red Hat Quay](https://www.redhat.com/en/technologies/cloud-computing/quay) container registry as a systemd-managed quadlet service stack on RHEL 9.

This repo is part of the [RHIS Builder](https://github.com/parmstro/rhis-builder-provisioner) project. All variables are configured through the unified inventory project [rhis-builder-inventory](https://github.com/parmstro/rhis-builder-inventory).

## Stack Components

| Service | Image | Description |
|---------|-------|-------------|
| quay | `registry.redhat.io/quay/quay-rhel9` | Registry application |
| quay-postgres | `registry.redhat.io/rhel9/postgresql-16` | Quay metadata database |
| quay-clair-postgres | `registry.redhat.io/rhel9/postgresql-16` | Clair vulnerability database |
| quay-redis | `registry.redhat.io/rhel9/redis-7` | Build logs and user events |
| quay-clair | `registry.redhat.io/quay/clair-rhel9` | Vulnerability scanner (combo mode) |
| quay-mirror | `registry.redhat.io/quay/quay-rhel9` | Repository mirroring worker |

## Architecture

The stack uses a pod for the Quay application containers and a bridge network for infrastructure services:

- **Pod** (`quay.pod`): Contains `quay` and `quay-mirror`. Publishes HTTP (8080) and HTTPS (8443) to the host.
- **Network** (`quay-network.network`): Infrastructure services (`quay-postgres`, `quay-clair-postgres`, `quay-redis`, `quay-clair`) run standalone on this bridge network.

Services resolve each other by container name via Podman DNS. No database, cache, or scanner ports are exposed to the host.

## Prerequisites

- RHEL 9 target host with Podman 4.4+ (quadlet support)
- Red Hat subscription with access to `registry.redhat.io`
- [rhis-builder-inventory](https://github.com/parmstro/rhis-builder-inventory) configured with Quay variables and vault secrets

## Deployment Phases

The `quay_deploy` role executes in six phases:

1. **Prerequisites** — installs container tools, authenticates to `registry.redhat.io`, opens firewall ports, creates config directories
2. **Generate configs** — renders `config.yaml`, `clair-config.yaml`, and `quay.env` from inventory variables and vault secrets
3. **Deploy quadlet files** — places `.pod`, `.container`, `.network`, and `.volume` unit files into `/etc/containers/systemd/` and reloads systemd
4. **Initialize database** — starts PostgreSQL, waits for readiness, installs the `pg_trgm` extension required by Quay
5. **Start services** — phased startup: PostgreSQL + Redis, then Quay (runs database migrations on first start), then Clair and mirror worker
6. **Validate** — checks the `/health/instance` endpoint and verifies all containers are running

## Usage

### With rhis-builder-inventory (standard)

```bash
ansible-playbook --inventory /path/to/inventory \
                 --ask-vault-password \
                 --extra-vars "vault_dir=/path/to/vault" \
                 --limit quay_servers \
                 main.yml
```

### Run a specific task

```bash
ansible-playbook --inventory /path/to/inventory \
                 --ask-vault-password \
                 --extra-vars "vault_dir=/path/to/vault" \
                 --extra-vars "role_name=quay_deploy" \
                 --extra-vars "task_name=validate_deployment" \
                 --limit quay_servers \
                 run_role_task.yml
```

## Inventory Configuration

### Required Variables (group_vars/quay_servers)

These must be provided via [rhis-builder-inventory](https://github.com/parmstro/rhis-builder-inventory):

| Variable | Source | Description |
|----------|--------|-------------|
| `quay_registry` | group_vars | Container registry (`registry.redhat.io`) |
| `quay_registry_username` | vault | Username for `registry.redhat.io` |
| `quay_registry_password` | vault | Password for `registry.redhat.io` |
| `quay_pg_password` | vault | Shared PostgreSQL password |
| `quay_database_secret_key` | vault | Encrypts sensitive fields in the database |
| `quay_session_secret_key` | vault | Secret key for session cookies and CSRF tokens |
| `quay_security_scanner_psk` | vault | Pre-shared key between Quay and Clair |

### Host-Level Variables (host_vars/quay1)

| Variable | Default | Description |
|----------|---------|-------------|
| `quay_server_hostname` | `quay1.<domain>` | FQDN clients use to reach the registry |
| `quay_http_port` | `8080` | HTTP port published to host |
| `quay_https_port` | `8443` | HTTPS port published to host |
| `quay_preferred_url_scheme` | `https` | `https` (with TLS certs) or `http` (with reverse proxy) |
| `quay_external_tls_termination` | `false` | Set `true` when TLS is handled by a reverse proxy |
| `quay_feature_security_scanner` | `true` | Enable Clair vulnerability scanning |
| `quay_feature_repo_mirror` | `true` | Enable repository mirroring |
| `quay_feature_mailing` | `false` | Enable email notifications |
| `quay_superusers` | (unset) | List of superuser account names |

### Vault Secrets

Add to your `rhis_builder_vault.yml`:

```yaml
quay_pg_password_vault: "<generate with: python3 -c 'import secrets; print(secrets.token_hex(16))'>"
quay_database_secret_key_vault: "<generate with: python3 -c 'import uuid; print(uuid.uuid4())'>"
quay_session_secret_key_vault: "<generate with: python3 -c 'import uuid; print(uuid.uuid4())'>"
quay_security_scanner_psk_vault: "<generate with: python3 -c 'import secrets; print(secrets.token_hex(32))'>"
```

The registry credentials (`containerhost_registry_username_vault` / `containerhost_registry_password_vault`) are shared with other RHIS container deployments and should already exist in your vault.

## TLS Configuration

**With TLS certificates** (default): Place `ssl.cert` and `ssl.key` in the Quay config directory (`/srv/containers/quay/config/`). Clients connect via HTTPS on port 8443.

**With reverse proxy**: Set `quay_preferred_url_scheme: "http"` and `quay_external_tls_termination: true` in host_vars. Add your proxy container to the `quay-backend` network and proxy to `quay:8080`.

**Standard ports**: To use ports 80/443 instead of 8080/8443, update `quay_http_port` and `quay_https_port` in host_vars. Binding to ports below 1024 requires running as root.

## Post-Deployment

After the stack is running, access the web UI at `https://<quay_server_hostname>:<quay_https_port>` to create an initial user account. Robot accounts for CI/push access can be created under User Settings or Organization settings.

Configure clients to access the registry:

```bash
podman login quay1.example.ca:8443
podman push quay1.example.ca:8443/org/image:tag
```

## Related Repositories

- [rhis-builder-inventory](https://github.com/parmstro/rhis-builder-inventory) — unified inventory and configuration
- [rhis-builder-quadlet-deploy](https://github.com/parmstro/rhis-builder-quadlet-deploy) — generic single-container quadlet deployer
- [rhis-provisioner-container](https://github.com/parmstro/rhis-provisioner-container) — containerized provisioner

PRs are always welcome.
