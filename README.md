# adem_abichou.komodo

Ansible Collection to deploy [Komodo](https://komo.do) — the open-source server management platform — on any Linux server.

## Roles

| Role | Description |
|---|---|
| `adem_abichou.komodo.komodo_core` | Deploys Komodo Core as a Docker Compose stack (FerretDB or MongoDB backend, optional Traefik) |
| `adem_abichou.komodo.komodo_periphery` | Installs the Komodo Periphery agent as a systemd binary service |

Each role has its own `README.md` with the full variable reference and examples:
- [roles/komodo_core/README.md](roles/komodo_core/README.md)
- [roles/komodo_periphery/README.md](roles/komodo_periphery/README.md)

---

## Requirements

- Ansible ≥ 2.14
- `community.docker` collection ≥ 3.0 (declared as a collection dependency — installed automatically with `ansible-galaxy collection install`)
- Docker + Compose plugin on the **Core** host (not managed by this collection — use `geerlingguy.docker` or equivalent)
- systemd on **Periphery** hosts

---

## Installation

```bash
ansible-galaxy collection install adem_abichou.komodo
# or pin a version:
ansible-galaxy collection install adem_abichou.komodo:==1.0.0
```

Install role and collection dependencies (needed for the example playbooks):

```bash
ansible-galaxy install -r requirements.yml
```

---

## Quick Start

### Minimal inventory

```yaml
# inventory/hosts.yml
all:
  children:
    komodo_core:
      hosts:
        my-core-server:
          ansible_host: 1.2.3.4
          ansible_user: ubuntu

    komodo_periphery:
      hosts:
        worker-1:
          ansible_host: 10.0.0.10
          ansible_user: ubuntu

  vars:
    # Required secrets — no defaults; role will fail fast if missing.
    # Generate: openssl rand -hex 32
    komodo_jwt_secret:          !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
    komodo_webhook_secret:      !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
    komodo_db_password:         !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
    komodo_init_admin_password: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...

    periphery_core_address: "ws://1.2.3.4:9120"
```

### Deploy Core only

```yaml
# playbook.yml
- hosts: komodo_core
  become: true
  roles:
    - role: geerlingguy.docker
    - role: adem_abichou.komodo.komodo_core
```

### Deploy Core + Periphery (full)

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

See [playbooks/](playbooks/) for ready-to-run example playbooks.

---

## Example Playbooks

| Playbook | Purpose |
|---|---|
| `playbooks/site.yml` | Full deployment: Docker → Core → API bootstrap → Periphery |
| `playbooks/core.yml` | Deploy Core only |
| `playbooks/periphery.yml` | Deploy Periphery agents only |
| `playbooks/bootstrap.yml` | Bootstrap Komodo API: create automation key + onboarding keys |

> `bootstrap.yml` must run after `core.yml` and before `periphery.yml` when using
> automatic server onboarding. It saves credentials to `.komodo_api_creds.json`
> and onboarding keys to `.komodo_onboarding_keys.json` locally.

---

## Upgrade Notes

To update Komodo after a new release:

```bash
# Update komodo_version in your group_vars or pass it on the command line:
ansible-playbook -i inventory/hosts.yml playbooks/site.yml \
  -e komodo_version=2.2.0 \
  --tags core,periphery
```

---

## Security Notes

- **Required secrets have no defaults.** The role asserts they are set and will fail with a clear message if not. Use `ansible-vault` to encrypt them.
- Periphery private keys are auto-generated on first boot and never leave the server.
- Onboarding keys are one-time use. Once a Periphery agent connects, the key can be discarded.
- Rate limiting is enabled on the Komodo Core API by default.

---

## Compatibility

| Komodo | Collection |
|---|---|
| 2.x | 1.x |

---

## License

MIT
