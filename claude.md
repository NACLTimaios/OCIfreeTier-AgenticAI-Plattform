# Agent Platform — Infrastructure (Ansible)

## Project
Configuration and deployment of the SQL Database Monitoring & Troubleshooting
Agent lab environment using Ansible. Goal: idempotent, reproducible setup of all
VMs from scratch, providing a safety net (re-provision + re-run converges back to
desired state).

## Environment — OCI Free Tier
VMs are provisioned manually in the OCI console (no Terraform). Ansible configures
them. All VMs share one VCN across two subnets. The fleet is expected to grow, so
everything must be group-driven and additive.

Current hosts:
- vm1  (E2.1.Micro, x86_64)   — PostgreSQL 16 database server
- vm2  (E2.1.Micro, x86_64)   — traffic generator (pgbench / sysbench)
- arm1 (Ampere A1, ARM64)     — agent backend: orchestrator, tool registry, HITL queue, log store
- arm2 (Ampere A1, ARM64)     — agent frontend: dashboard, HITL approval panel, query interface

## Constraints
- MIXED ARCHITECTURE: vm1/vm2 are x86_64, arm1/arm2 are ARM64. Roles that install
  arch-specific packages (Docker, binaries) MUST detect architecture via the
  `ansible_architecture` fact and select the correct repo/asset. Never hardcode arm64.
- Verify any container image used has both amd64 and arm64 variants, or pin per-group.
- Keep memory footprint modest (Micro = 1GB; Ampere = up to 12GB per instance).

## Repository Layout
```
agent-platform-infra/
├── ansible.cfg
├── inventory/hosts.yml          # all hosts grouped by role
├── group_vars/                  # all.yml + per-group vars
├── roles/                       # common, docker, claude_code, postgres,
│                                #   traffic_gen, agent_backend, agent_frontend
├── playbooks/                   # site.yml, common.yml, verify.yml
└── README.md
```

## Inventory Groups
Map roles to groups, not to individual hosts. Adding a VM later must be a one-line
inventory change that inherits the full role automatically.
- db              -> vm1
- traffic         -> vm2
- agent_backend   -> arm1
- agent_frontend  -> arm2

## Where Claude Code Runs
The `claude_code` role is applied ONLY to arm1 (or a dedicated management host if
one is added). The other VMs are managed targets — Ansible configures them from the
control machine; they do not need Claude Code installed.

arm1 is BOTH the Ansible control node and a managed host (agent_backend). It
configures itself, so in the inventory arm1 uses `ansible_connection: local` —
Ansible acts on arm1's own filesystem directly rather than over SSH. The other
three hosts (vm1, vm2, arm2) are reached over SSH from arm1 using the control
node's key.

## Context Gathering
Before writing or changing anything, gather context yourself — do not guess and do
not ask the user for things you can find on your own.

- Read existing roles/playbooks/vars before editing; match current structure and
  naming conventions before adding to them
- Inspect the repo layout (tree, ls) to locate the right role/task file rather than
  assuming a path
- For Ansible modules, collection names, and arguments you are unsure about, verify
  current usage via WebSearch / WebFetch rather than relying on memory — module
  names and options change between Ansible/collection versions (e.g. community.docker,
  community.postgresql, ansible.posix)
- Check declared collection and role versions (requirements.yml, ansible.cfg) and
  match tasks to those versions
- When verifying package or image availability per architecture, look it up rather
  than assuming
- Only ask the user when a decision is genuinely ambiguous and cannot be resolved
  from the repo, the docs, or a search — otherwise proceed and state the assumption

## Ansible Conventions
- IDEMPOTENT ONLY: every task must be safe to run repeatedly. Re-running a playbook
  must result in no changes when the system is already in the desired state.
- Prefer modules over shell/command. Only use shell/command when no module exists,
  and when you do, set `changed_when` / `creates` / `args` so the task reports
  changed state correctly and stays idempotent.
- Use handlers for service restarts/reloads — never restart unconditionally inside a task.
- Roles are self-contained: defaults/main.yml for overridable vars, tasks/main.yml
  for logic, handlers/main.yml for handlers, templates/ for Jinja2.
- Configuration values (versions, CIDRs, ports, credentials references) live in
  group_vars/ — never hardcoded in tasks.
- Secrets handled per the Secrets section below — never plaintext in the repo.
- Name every task descriptively. Tag tasks by role/purpose so partial runs are possible.
- Architecture-aware: branch on `ansible_architecture` where packages differ.
- Network: OCI Ubuntu images ship restrictive iptables defaults — the common role
  must allow intra-VCN traffic on the configured CIDR and persist the rule.
- Minimal images: targets use Ubuntu 22.04 Minimal, which may ship without
  python3. The common role's FIRST task must bootstrap Python via the `raw`
  module (gather_facts deferred until after), since Ansible's standard modules
  require a Python interpreter. Use `test -e /usr/bin/python3 || (apt-get update
  && apt-get install -y python3)` with `changed_when: false`, then re-run setup
  to gather facts.

## Secrets
This is a test lab, so secrets use Ansible Vault with a password file — built in,
no external service, but never plaintext in the repo. Convention:

- Encrypted secrets live in `group_vars/all/vault.yml`, with every key prefixed
  `vault_` (e.g. `vault_postgres_password`).
- A plain (unencrypted) `group_vars/all/vars.yml` maps each to a clean alias:
  `postgres_password: "{{ vault_postgres_password }}"`.
- Roles ONLY ever reference the plain alias (`postgres_password`), never the
  `vault_`-prefixed key directly. This keeps the secret source swappable — Vault
  today, a real secret manager later — without touching any role.
- The vault password is read from a gitignored `.vault_pass` file, wired via
  `vault_password_file = .vault_pass` in `ansible.cfg`, so no `--ask-vault-pass`
  prompt is needed on each run.
- `.gitignore` MUST include `.vault_pass`. The encrypted `vault.yml` is safe to
  commit; the password file is not.
- Generated READMEs must note: this is convenience-grade for a test lab. If the
  environment becomes real or internet-facing, rotate credentials and migrate to a
  proper secret manager.

## Self Tests / Verification
Verification is part of the deliverable, not an afterthought. A failed verify run
must exit non-zero.

- Provide `playbooks/verify.yml` that checks each role's outcome:
  - common: required packages present, docker group exists, iptables rule present
  - docker: `docker` service active, `docker run --rm hello-world` succeeds
    (correct arch image)
  - postgres (db group): service active, port reachable from agent subnet,
    pg_stat_statements loaded
  - traffic_gen: pgbench/sysbench binaries present and runnable
  - agent_backend / agent_frontend: expected containers up, health endpoints
    return 200
- Verification must use Ansible's own assertions (assert, command with failed_when,
  uri module for HTTP health checks) — not real LLM calls or external network.
- Use `--check` (dry-run) compatibility: tasks should support check mode where
  feasible so a plan can be previewed before applying.
- Document in the README how to run a full deploy, a per-group deploy (via --limit),
  a dry-run (--check), and the verify playbook.

## Definition of Done
No role is complete until:
1. It is idempotent (a second run reports zero changes).
2. Its checks exist in verify.yml and pass.
3. Its overridable values live in group_vars/defaults, not hardcoded in tasks.
When adding or modifying a role, update verify.yml and the README in the same change.

## Tooling
ansible-core is **not** installed system-wide. It is managed via **pipx**:

```
pipx install ansible==10.7.0   # bundles ansible-core 2.17.14
```

The `ansible` binary lives in `~/.local/bin` (added to PATH by pipx). Collections
are installed into `~/.ansible/collections` with `ansible-galaxy collection install
-r requirements.yml`. The currently installed (pinned) collection versions are:

| Collection            | Version |
|-----------------------|---------|
| ansible.posix         | 2.2.0   |
| community.docker      | 5.2.1   |
| community.general     | 13.0.1  |
| community.postgresql  | 4.2.0   |

`requirements.yml` declares minimum floor versions (`>=`); the table above is what
is actually installed on the control node. When writing tasks, target the installed
versions — don't assume features from newer releases are available.

## How It's Run
From a control machine with SSH access to all private IPs:
```
ansible-playbook -i inventory/hosts.yml playbooks/site.yml          # full deploy
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --check  # dry-run
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --limit db
ansible-playbook -i inventory/hosts.yml playbooks/verify.yml        # verify
```
