# komodo_periphery

Installs and configures [Komodo Periphery](https://komo.do) — the worker agent — as a systemd service.

The binary is downloaded from the official GitHub release assets (`moghtech/komodo`).
Supports `x86_64` and `aarch64` architectures.

Periphery runs in **outbound mode**: it initiates a WebSocket connection to Core,
so no inbound port needs to be opened on the Periphery host.

---

## Requirements

- systemd-based Linux host
- Network access to the Komodo Core WebSocket address (`ws://` or `wss://`)
- Ansible ≥ 2.14
- `become: true` (the role installs a binary to `/usr/local/bin` and a systemd unit)

Docker is **not** required on the Periphery host for the agent itself, but Komodo
will need it to manage containers — install it separately if needed.

---

## Role Variables

### Binary

| Variable | Default | Description |
|---|---|---|
| `komodo_version` | `2.1.1` | Komodo release tag to download |
| `periphery_binary_url` | *(auto)* | Full URL to the release binary. Auto-built from `komodo_version` + arch. Override to use a mirror or pre-downloaded path |
| `periphery_binary_path` | `/usr/local/bin/periphery` | Install path for the binary |

### Runtime user

| Variable | Default | Description |
|---|---|---|
| `periphery_user` | `root` | User that runs the systemd service |
| `periphery_group` | `root` | Group for the systemd service |

> Periphery needs access to the Docker socket. Ensure the user is in the `docker`
> group (or use `root`) for full functionality.

### Directories

| Variable | Default | Description |
|---|---|---|
| `periphery_root_directory` | `/etc/komodo` | Root directory for Periphery data (repos, stacks, builds) |
| `periphery_config_dir` | `/etc/komodo` | Directory for the config file |
| `periphery_config_path` | `/etc/komodo/periphery.config.toml` | Full path to the config file |

### Connection (outbound to Core)

| Variable | Default | Description |
|---|---|---|
| `periphery_core_address` | `ws://localhost:9120` | WebSocket URL of Komodo Core. Use `wss://` for TLS |
| `periphery_connect_as` | `{{ inventory_hostname }}` | Name this Periphery registers/connects as in Core. Must match an existing Server or onboarding creates one |
| `periphery_onboarding_key` | `""` | One-time key for self-registration as a new Server. Leave empty if the Server already exists in Core. Set by the `init.yml` playbook automatically |

### Feature flags

| Variable | Default | Description |
|---|---|---|
| `periphery_disable_terminals` | `false` | Disable the terminal feature in Komodo UI |
| `periphery_disable_container_terminals` | `false` | Disable container exec terminals |
| `periphery_legacy_compose_cli` | `false` | Use `docker-compose` v1 CLI instead of the Compose plugin |

### Disk reporting

| Variable | Default | Description |
|---|---|---|
| `periphery_include_disk_mounts` | `["/etc/hostname"]` | Paths to include when reporting disk usage. `/etc/hostname` gives accurate host disk size |
| `periphery_exclude_disk_mounts` | `[]` | Paths to exclude from disk reporting |

### Poll intervals

| Variable | Default | Options |
|---|---|---|
| `periphery_stats_polling_rate` | `5-sec` | See [Timelength docs](https://docs.rs/komodo_client/latest/komodo_client/entities/enum.Timelength.html) |
| `periphery_container_stats_polling_rate` | `30-sec` | Same options |

### Logging

| Variable | Default | Description |
|---|---|---|
| `periphery_logging_level` | `info` | `off`, `error`, `warn`, `info`, `debug`, `trace` |
| `periphery_logging_stdio` | `standard` | `standard`, `json`, `none` |
| `periphery_logging_pretty` | `false` | Pretty-print log output |
| `periphery_pretty_startup_config` | `false` | Log full config on startup |

### Secrets (optional)

| Variable | Default | Description |
|---|---|---|
| `periphery_secrets` | `{}` | Key/value pairs exposed as secrets to stacks managed by this agent |

Example:
```yaml
periphery_secrets:
  REGISTRY_TOKEN: "dckr_pat_..."
  DEPLOY_KEY: "{{ vault_deploy_key }}"
```

---

## Dependencies

None. Docker on the target host is optional (needed only for container management).

---

## Example Playbooks

### Minimal — connect to Core by IP

```yaml
- hosts: worker_nodes
  become: true
  vars:
    periphery_core_address: "ws://10.0.0.4:9120"
  roles:
    - role: komodo_periphery
```

The Periphery agent connects to Core on that address. If a Server with the host's
`inventory_hostname` already exists in Core it will connect; otherwise, set
`periphery_onboarding_key` to self-register.

---

### Multi-node with onboarding keys

When using the `init.yml` bootstrap playbook, onboarding keys are generated
automatically. You can also set them directly in inventory for a manual setup:

```yaml
all:
  children:
    komodo_periphery:
      hosts:
        worker-1:
          ansible_host: 10.0.0.10
          ansible_user: ubuntu
          periphery_onboarding_key: "{{ vault_worker1_onboarding_key }}"
        worker-2:
          ansible_host: 10.0.0.11
          ansible_user: ubuntu
          periphery_onboarding_key: "{{ vault_worker2_onboarding_key }}"
  vars:
    periphery_core_address: "ws://10.0.0.4:9120"
```

---

### Behind Traefik (Core exposed via TLS)

```yaml
vars:
  periphery_core_address: "wss://komodo.example.com"
  periphery_connect_as: "worker-eu-1"
```

---

### Single-node colocation (Core + Periphery on same host)

```yaml
- hosts: single_node
  become: true
  vars:
    # Core vars
    komodo_jwt_secret: "{{ vault_jwt }}"
    komodo_webhook_secret: "{{ vault_webhook }}"
    komodo_db_password: "{{ vault_db }}"
    komodo_init_admin_password: "{{ vault_admin }}"
    komodo_core_add_host_gateway: true
    komodo_first_server_name: "local"
    # Setting only the name (no address) = outbound Periphery mode — Core waits for Periphery to connect
    # Periphery vars (no onboarding key needed — Core auto-registers via first_server_name)
    periphery_core_address: "ws://localhost:9120"
  roles:
    - role: geerlingguy.docker
    - role: komodo_core
    - role: komodo_periphery
```

---

### With per-agent secrets

```yaml
vars:
  periphery_core_address: "ws://10.0.0.4:9120"
  periphery_secrets:
    DOCKER_HUB_TOKEN: "{{ vault_dockerhub_token }}"
    GITHUB_TOKEN: "{{ vault_github_token }}"
```

---

## Inventory Example

```yaml
all:
  children:
    komodo_periphery:
      hosts:
        worker-1:
          ansible_host: 10.0.0.10
          ansible_user: ubuntu
        worker-2:
          ansible_host: 10.0.0.11
          ansible_user: ubuntu
  vars:
    periphery_core_address: "ws://10.0.0.4:9120"
    # onboarding key set per-host in host_vars/ or generated by init.yml
```

---

## How auto-registration works

1. Run `komodo_core` role to deploy Core
2. Run `playbooks/init.yml` (or the equivalent API calls) to create per-host onboarding keys
3. Run `komodo_periphery` role with `periphery_onboarding_key` set — Periphery
   self-registers as a new Server in Core and the key is consumed

On subsequent runs `periphery_onboarding_key` can be left empty — Periphery
will reconnect as the already-registered Server using `periphery_connect_as`.

---

## License

MIT
