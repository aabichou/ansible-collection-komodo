# komodo_core

Deploys [Komodo Core](https://komo.do) — the central management server — as a Docker Compose stack.

Supported database backends:
- **FerretDB** (default) — MongoDB-compatible API on top of PostgreSQL; no AVX CPU requirement
- **MongoDB** — native MongoDB; requires AVX CPU support

Optional Traefik reverse-proxy integration (TLS termination, routing by subdomain).

---

## Requirements

- Docker + Docker Compose plugin installed on the target host
  (use [`geerlingguy.docker`](https://github.com/geerlingguy/ansible-role-docker) or your own role)
- `community.docker` Ansible collection ≥ 3.0 (`ansible-galaxy collection install community.docker`)
- Ansible ≥ 2.14

---

## Role Variables

### Required — no defaults, role will fail fast if missing

| Variable | Description |
|---|---|
| `komodo_jwt_secret` | JWT signing secret. Generate: `openssl rand -hex 32` |
| `komodo_webhook_secret` | Git webhook HMAC secret. Generate: `openssl rand -hex 32` |
| `komodo_db_password` | Database password (MongoDB / FerretDB) |
| `komodo_init_admin_password` | Initial admin UI password |

### Core settings

| Variable | Default | Description |
|---|---|---|
| `komodo_version` | `2.1.1` | Komodo image tag to deploy |
| `komodo_core_dir` | `/opt/komodo` | Compose project root on the host |
| `komodo_core_backups_path` | `/etc/komodo/backups` | Host path for backups volume |
| `komodo_core_port` | `9120` | Host port the Core container listens on |
| `komodo_core_public_url` | `http://<host-ip>:9120` | Public URL used for OAuth callbacks and webhook hints |
| `komodo_title` | `Komodo` | Browser tab / UI title |
| `komodo_timezone` | `Etc/UTC` | TZ identifier (e.g. `Europe/Paris`) |

### Database

| Variable | Default | Description |
|---|---|---|
| `komodo_db_type` | `ferretdb` | `ferretdb` or `mongo` |
| `komodo_db_name` | `komodo` | Database / schema name |
| `komodo_db_username` | `komodo` | Database username |

### Authentication

| Variable | Default | Description |
|---|---|---|
| `komodo_local_auth` | `true` | Enable username/password login |
| `komodo_init_admin_username` | `admin` | Admin account username created on first boot |
| `komodo_disable_user_registration` | `false` | Prevent new user sign-ups |
| `komodo_enable_new_users` | `false` | Auto-enable newly registered users |
| `komodo_disable_non_admin_create` | `false` | Restrict resource creation to admins |
| `komodo_transparent_mode` | `false` | Let all users see all resources |
| `komodo_disable_confirm_dialog` | `false` | Skip destructive-action confirmation dialogs |
| `komodo_disable_init_resources` | `false` | Skip default resource creation on first boot |
| `komodo_periphery_public_key` | `""` | Pin Periphery public key Core will accept (Noise PKI). Empty = accept any |

### JWT

| Variable | Default | Description |
|---|---|---|
| `komodo_jwt_ttl` | `1-day` | JWT lifetime (`1-hr`, `12-hr`, `1-day`, `3-day`, `1-wk`, `2-wk`) |

### First server (quick single-node shortcut)

| Variable | Default | Description |
|---|---|---|
| `komodo_first_server_address` | `""` | Auto-register first periphery on Core boot (e.g. `ws://localhost:8120`). Leave empty when using the `init.yml` bootstrap playbook |
| `komodo_first_server_name` | `Local` | Display name for the first server. **TIP:** setting only the name (no address) tells Core to expect an outbound Periphery connection — the recommended single-node pattern |

### Poll intervals

| Variable | Default | Options |
|---|---|---|
| `komodo_monitoring_interval` | `15-sec` | `1-sec`, `5-sec`, `15-sec`, `1-min`, `5-min`, `15-min` |
| `komodo_resource_poll_interval` | `1-hr` | `15-min`, `1-hr`, `2-hr`, `6-hr`, `12-hr`, `1-day` |

### Pruning

| Variable | Default | Description |
|---|---|---|
| `komodo_keep_stats_for_days` | `14` | Days to retain monitoring stats |
| `komodo_keep_alerts_for_days` | `14` | Days to retain alert history |

### Logging

| Variable | Default | Description |
|---|---|---|
| `komodo_logging_level` | `info` | `off`, `error`, `warn`, `info`, `debug`, `trace` |
| `komodo_logging_pretty` | `false` | Pretty-print log output |
| `komodo_pretty_startup_config` | `false` | Log full config on startup |

### Template overrides

| Variable | Default | Description |
|---|---|---|
| `komodo_core_compose_template` | `compose.yml.j2` | Jinja2 template for the Docker Compose file. Override with an absolute path or a path relative to your playbook's `templates/` directory |
| `komodo_core_env_template` | `compose.env.j2` | Jinja2 template for the compose `.env` file. Same path rules as above |

### Networking

| Variable | Default | Description |
|---|---|---|
| `komodo_core_add_host_gateway` | `false` | Add `host.docker.internal` extra_host. Set `true` when Periphery (systemd) runs on the same host as Core (Docker) |
| `komodo_core_containerized_periphery` | `false` | Set `true` when running Periphery as a container on the same host as Core. Adds `PERIPHERY_*` vars to the env file |

### Traefik integration (optional)

| Variable | Default | Description |
|---|---|---|
| `traefik_enabled` | `false` | Attach Traefik labels to the Core container and join the external network |
| `traefik_external_tld` | `""` | Root domain (e.g. `example.com`) |
| `traefik_core_subdomain` | `komodo` | Subdomain for Core (results in `komodo.example.com`) |

### OIDC (optional)

| Variable | Default | Description |
|---|---|---|
| `komodo_oidc_enabled` | `false` | Enable OIDC login |
| `komodo_oidc_provider` | `""` | OIDC issuer URL |
| `komodo_oidc_redirect_host` | `""` | Redirect URI base (usually same as `komodo_core_public_url`) |
| `komodo_oidc_client_id` | `""` | OIDC client ID |
| `komodo_oidc_client_secret` | `""` | OIDC client secret |
| `komodo_oidc_use_full_email` | `false` | Use full email as Komodo username |

### GitHub OAuth (optional)

| Variable | Default | Description |
|---|---|---|
| `komodo_github_oauth_enabled` | `false` | Enable GitHub OAuth login |
| `komodo_github_oauth_id` | `""` | GitHub OAuth App client ID |
| `komodo_github_oauth_secret` | `""` | GitHub OAuth App client secret |

### Google OAuth (optional)

| Variable | Default | Description |
|---|---|---|
| `komodo_google_oauth_enabled` | `false` | Enable Google OAuth login |
| `komodo_google_oauth_id` | `""` | Google OAuth client ID |
| `komodo_google_oauth_secret` | `""` | Google OAuth client secret |

---

## Dependencies

None declared. This role requires Docker to already be installed on the target host.
Use [`geerlingguy.docker`](https://github.com/geerlingguy/ansible-role-docker) or manage Docker separately.

---

## Example Playbooks

### Minimal — direct IP, no reverse proxy

```yaml
- hosts: komodo_servers
  become: true
  vars:
    komodo_jwt_secret: "{{ vault_komodo_jwt_secret }}"
    komodo_webhook_secret: "{{ vault_komodo_webhook_secret }}"
    komodo_db_password: "{{ vault_komodo_db_password }}"
    komodo_init_admin_password: "{{ vault_komodo_init_admin_password }}"
  roles:
    - role: geerlingguy.docker
    - role: komodo_core
```

Access Komodo at `http://<server-ip>:9120`.

---

### Single-node with auto-registered local Periphery

When Core and Periphery run on the same host, you can skip the init bootstrap
and let Core auto-register the local Periphery:

```yaml
- hosts: komodo_server
  become: true
  vars:
    komodo_jwt_secret: "{{ vault_komodo_jwt_secret }}"
    komodo_webhook_secret: "{{ vault_komodo_webhook_secret }}"
    komodo_db_password: "{{ vault_komodo_db_password }}"
    komodo_init_admin_password: "{{ vault_komodo_init_admin_password }}"
    # Core talks to the systemd Periphery on the same host via host-gateway
    komodo_core_add_host_gateway: true
    komodo_first_server_name: "local"
    # Setting only the name (no address) = outbound Periphery mode; no FIRST_SERVER_ADDRESS needed
  roles:
    - role: geerlingguy.docker
    - role: komodo_core
    - role: komodo_periphery
```

---

### Behind Traefik with TLS

```yaml
- hosts: komodo_core
  become: true
  vars:
    komodo_jwt_secret: "{{ vault_komodo_jwt_secret }}"
    komodo_webhook_secret: "{{ vault_komodo_webhook_secret }}"
    komodo_db_password: "{{ vault_komodo_db_password }}"
    komodo_init_admin_password: "{{ vault_komodo_init_admin_password }}"
    komodo_core_public_url: "https://komodo.example.com"
    traefik_enabled: true
    traefik_external_tld: "example.com"
    traefik_core_subdomain: "komodo"
  roles:
    - role: komodo_core
```

> Traefik must be running and the external Docker network must exist before this play runs.

---

### Custom Compose templates

You can supply your own `compose.yml.j2` and `compose.env.j2` templates to
customise the Docker Compose stack beyond what the role variables expose:

```yaml
- hosts: komodo_core
  become: true
  vars:
    komodo_core_compose_template: "{{ playbook_dir }}/templates/my-compose.yml.j2"
    komodo_core_env_template: "{{ playbook_dir }}/templates/my-compose.env.j2"
  roles:
    - role: komodo_core
```

The built-in templates (in the role's `templates/` directory) are a good starting
point — copy them into your project and modify as needed.

---

### With OIDC (e.g. Keycloak)

```yaml
vars:
  komodo_core_public_url: "https://komodo.example.com"
  komodo_oidc_enabled: true
  komodo_oidc_provider: "https://keycloak.example.com/realms/myrealm"
  komodo_oidc_redirect_host: "https://komodo.example.com"
  komodo_oidc_client_id: "komodo"
  komodo_oidc_client_secret: "{{ vault_oidc_secret }}"
  # local_auth can be disabled once OIDC is confirmed working
  komodo_local_auth: false
```

---

## Inventory Example

```yaml
all:
  children:
    komodo_core:
      hosts:
        my-core-server:
          ansible_host: 1.2.3.4
          ansible_user: ubuntu
  vars:
    komodo_jwt_secret: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
    komodo_webhook_secret: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
    komodo_db_password: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
    komodo_init_admin_password: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
```

Encrypt secrets with `ansible-vault encrypt_string 'value' --name varname`.

---

## License

MIT
