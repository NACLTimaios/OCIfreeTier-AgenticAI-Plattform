# Agent Platform — Infrastructure (Ansible)

Idempotent Ansible configuration for the SQL Database Monitoring & Troubleshooting Agent
lab environment on OCI Free Tier. Re-running any playbook converges the system back to
the desired state with zero changes when everything is already correct.

## Architecture

| Host | Shape | Arch | Groups | Roles |
|------|-------|------|--------|-------|
| vm1  | E2.1.Micro | x86_64 | `db` | common, postgres |
| vm2  | E2.1.Micro | x86_64 | `traffic` | common, traffic_gen |
| arm1 | Ampere A1  | ARM64  | `agent_backend` | common, docker, claude_code, agent_backend |
| arm2 | Ampere A1  | ARM64  | `agent_frontend` | common, docker, agent_frontend |

arm1 is both the **Ansible control node** and a managed host.
It configures itself via `ansible_connection: local`; all other hosts are reached over SSH.

## Repository Layout

```
agent-platform-infra/
├── ansible.cfg                  # vault_password_file, fact_caching, pipelining
├── .gitignore                   # excludes .vault_pass
├── requirements.yml             # Galaxy collections
├── inventory/
│   ├── hosts.yml                # all hosts grouped by role
│   └── group_vars/              # adjacent to hosts.yml so Ansible resolves vars
│       ├── all/                 #   in both ad-hoc and ansible-playbook contexts
│       │   ├── vars.yml         #   clean aliases + shared vars
│       │   └── vault.yml        #   ENCRYPT THIS — contains vault_ prefixed secrets
│       ├── db/vars.yml
│       ├── traffic/vars.yml
│       ├── agent_backend/vars.yml
│       └── agent_frontend/vars.yml
├── roles/
│   ├── common/                  # Python bootstrap, packages, iptables
│   ├── docker/                  # Docker CE (arch-aware), compose plugin
│   ├── claude_code/             # Node.js + Claude Code CLI + API key
│   ├── postgres/                # PostgreSQL 16, pg_stat_statements
│   ├── traffic_gen/             # pgbench + sysbench
│   ├── agent_backend/           # Docker Compose: orchestrator, tool-registry, hitl-queue, log-store
│   └── agent_frontend/          # Docker Compose: dashboard, hitl-panel, query-interface
└── playbooks/
    ├── site.yml                 # full fleet deploy
    ├── common.yml               # bootstrap only
    └── verify.yml               # post-deploy assertions
```

## Prerequisites

### Control node (arm1)

Ansible is managed via **pipx** to keep it isolated from the system Python:

```bash
# Install pinned Ansible (bundles ansible-core 2.17.14)
pipx install ansible==10.7.0

# Install required collections into ~/.ansible/collections
ansible-galaxy collection install -r requirements.yml
```

### Vault password file

```bash
# Create and populate (gitignored)
echo 'your-vault-password' > .vault_pass
chmod 600 .vault_pass
```

### Encrypt secrets

```bash
# vault.yml ships as plaintext template — encrypt before first commit
ansible-vault encrypt inventory/group_vars/all/vault.yml

# Edit secrets
ansible-vault edit inventory/group_vars/all/vault.yml
```

> **Lab-only security notice:** Ansible Vault with a local `.vault_pass` file is
> convenience-grade for a test lab. If this environment becomes internet-facing or
> production, rotate all credentials and migrate to a proper secret manager
> (OCI Vault, HashiCorp Vault, etc.).

## Running

### Full deploy

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### Dry-run (check mode — no changes applied)

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check
```

> **Expected check-mode noise:** the `raw` Python bootstrap task is not
> check-mode aware and may warn on a clean host. Tasks that depend on
> earlier ones (e.g. file-permission tasks after a GPG dearmor) may show
> cascading failures. Both resolve on a real run.


### Per-group deploy

```bash
# Database server only
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --limit db

# Agent hosts only
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --limit agent_backend:agent_frontend

# Traffic generator only
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --limit traffic
```

### Per-tag deploy

Tags align with role names and sub-tasks:

```bash
# Apply only Docker tasks
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags docker

# Apply only firewall tasks across all hosts
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags firewall
```

### Bootstrap only (new VM, python not yet installed)

```bash
ansible-playbook -i inventory/hosts.yml playbooks/common.yml
```

### Verify

```bash
# Full verify
ansible-playbook -i inventory/hosts.yml playbooks/verify.yml

# Per-role verify
ansible-playbook -i inventory/hosts.yml playbooks/verify.yml --tags postgres
ansible-playbook -i inventory/hosts.yml playbooks/verify.yml --tags docker
```

## Roles

### common
- Bootstraps Python 3 via `raw` (works on Ubuntu Minimal with no Python installed).
- Installs common packages (curl, git, htop, iptables-persistent, …).
- Ensures the `docker` group exists.
- Removes the OCI default `REJECT`-all iptables rule (which sits ahead of any
  appended `ACCEPT` rules and blocks intra-VCN traffic). OCI security lists
  provide the outer perimeter.
- Opens intra-VCN traffic (`10.0.0.0/16`) via iptables and persists rules with
  `netfilter-persistent`.

### docker
- Detects host architecture (`ansible_architecture`) and selects the correct Docker
  apt repo (`amd64` vs `arm64`).
- Installs Docker CE + Compose plugin.
- Adds `ubuntu` to the `docker` group.

### claude_code
- Installs Node.js 20 via NodeSource (arm64-compatible).
- Installs `@anthropic-ai/claude-code` globally via npm.
- Writes `ANTHROPIC_API_KEY` to `/etc/environment`.

### postgres
- Adds the official PGDG apt repository.
- Installs PostgreSQL 16.
- Deploys `conf.d/00_ansible.conf` with tuned settings for 1 GB RAM.
- Deploys `pg_hba.conf` granting VCN subnet access to the application database.
- Creates the application user and database.
- Enables the `pg_stat_statements` extension.

### traffic_gen
- Adds the PGDG repo and installs `postgresql-16` and `postgresql-client-16`.
  (`pgbench` ships in the server package, not the client package.)
- Stops and **masks** the local PostgreSQL service — the PGDG installer uses a
  systemd generator that re-enables the unit at runtime, so `enabled: false`
  alone is not idempotent; `masked: true` is.
- Installs `sysbench` from Ubuntu's default repo.
- Creates a `/usr/local/bin/pgbench` symlink so it's on `PATH`.

### agent_backend
- Deploys a Docker Compose stack with four services: `orchestrator` (8080),
  `tool-registry` (8081), `hitl-queue` (8082), `log-store` (8083).
- Images default to `nginx:alpine` placeholders; override via `inventory/group_vars/agent_backend/vars.yml`.

### agent_frontend
- Deploys a Docker Compose stack with three services: `dashboard` (3000),
  `hitl-panel` (3001), `query-interface` (3002).
- Images default to `nginx:alpine` placeholders; override via `inventory/group_vars/agent_frontend/vars.yml`.

## Adding a New Host

1. Add the host to the appropriate group in `inventory/hosts.yml`.
2. Re-run `playbooks/site.yml` — it will configure the new host automatically.

## Definition of Done (per role)

A role is complete when:
1. A second run reports **zero changes** (idempotent).
2. Its checks exist in `verify.yml` and pass.
3. All overridable values live in `group_vars/`, not hardcoded in tasks.
