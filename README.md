# infra-orchestrator

Ansible-based orchestrator for managing home infrastructure without Docker — Raspberry Pi nodes and anything else that speaks SSH. Designed to expand to additional device types (routers, microcontrollers, etc.) as needs grow.

---

## Repository layout

```
infra-orchestrator/
├── ansible.cfg              # Global Ansible settings
├── inventory.ini            # Host groups (no IPs — resolved from vault)
├── requirements.yml         # Community roles
├── main.yml                 # Master playbook
├── tasks/
│   ├── toolchain.yml        # Build toolchain provided to pipelines
│   └── service_slot.yml     # Everything one service needs from the host
├── templates/               # Toolchain manifest, service unit, env, sudoers, route
├── handlers/
│   └── apache.yml           # Config test, then reload
├── group_vars/
│   └── all/
│       ├── vault.yml        # AES-256 encrypted secrets  ← MUST be encrypted
│       └── vars.yml         # Silent bridge (safe to commit — no real data)
└── host_vars/
    └── pi-server-1.yml      # Resolves ansible_host from vault
```

This repository uses a **zero-knowledge configuration**: even with full read access to the repo, an observer learns nothing about usernames, IPs, ports, or credentials without the vault password.

---

## Prerequisites

| Tool | Version |
|---|---|
| Ansible | ≥ 2.14 |
| ansible-galaxy | bundled with Ansible |
| sshpass | any |

```bash
sudo apt install ansible-core sshpass
```

---

## First-time server bootstrap (manual)

Before Ansible can connect, a dedicated `ansible` user must be created on each target Unix-like device. Do this once over direct access or any existing administrative account.

### 1 — Create the automation user

```bash
sudo useradd --create-home --shell /bin/bash --password '!' <username>
```

- `--password '!'` locks password login — SSH key is the only way in
- `/bin/bash` is required so Ansible can execute commands remotely

### 2 — Add your SSH public key

On your control machine, generate a key if you don't have one yet:

```bash
ssh-keygen -t ed25519
```

Then copy it to the target device in one command (run from your control machine):

```bash
cat ~/.ssh/id_ed25519.pub | ssh <admin-user>@<device-ip> "sudo mkdir -p /home/<username>/.ssh && sudo tee /home/<username>/.ssh/authorized_keys && sudo chmod 700 /home/<username>/.ssh && sudo chmod 600 /home/<username>/.ssh/authorized_keys && sudo chown -R <username>:<username> /home/<username>/.ssh"
```

### 3 — Grant passwordless sudo

```bash
sudo visudo -f /etc/sudoers.d/<username>
```

Add this line:

```
<username> ALL=(ALL) NOPASSWD: ALL
```

This is a full root grant. Provisioning installs packages, writes systemd units and creates users, so the automation account needs broad privileges — narrowing this list breaks plays in ways that surface as confusing sudo errors much later.

Because it is a full grant, nothing that executes untrusted code may run as this user. GitHub Actions workflows in particular run as a separate, unprivileged account — see below.

### 3b — The GitHub Actions runner account

Do **not** create this account by hand. `main.yml` creates it via the `Ensure the GitHub Actions runner user exists` pre-task, with:

- `/usr/sbin/nologin` shell and a locked password — no interactive login
- no SSH key — unreachable from outside
- no general sudo — the only entries in `/etc/sudoers.d/` are the ones each [service slot](#service-slots) creates, each allowing exactly `systemctl restart <service>`

Its username comes from `vault_github_runner_user`. Workflow jobs execute as this account, so it should only ever own the specific deployment directory a project needs. If a deploy has to restart a service, add a sudoers entry naming that exact command — never a general grant.

### 4 — Verify access from your control machine

```bash
ssh -i ~/.ssh/id_ed25519 <username>@<device-ip>
```

Should connect without a password prompt. Once confirmed, Ansible is ready to use.

---

## First-time setup

### 1 — Install required roles

```bash
ansible-galaxy install -r requirements.yml
```

### 2 — Populate and encrypt secrets

Variables are split across two files:

- **`group_vars/all/vault.yml`** — sensitive values (credentials, IPs, ports). Every variable here uses a `vault_` prefix. Fill in all placeholder values, then encrypt the file before committing:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

To edit it later:

```bash
ansible-vault edit group_vars/all/vault.yml
```

- **`group_vars/all/vars.yml`** — non-sensitive values (repo name, timezone, etc.) committed in plain text, plus a bridge mapping each `vault_*` variable to the name Ansible and roles expect. Safe to commit as-is.

---

## Running playbooks

### Full provisioning

```bash
ansible-playbook main.yml
```

> Ansible will prompt for your vault password on each run.

### Target a specific host

```bash
ansible-playbook main.yml --limit pi-server-1
```

### Dry-run (check mode)

```bash
ansible-playbook main.yml --check --diff
```

---

## Security notes

* **`group_vars/all/vault.yml` must be AES-256 encrypted** before any commit — see step 2 above.
* Ansible prompts for the vault password on each run — no plaintext password file is used or needed.
* `group_vars/all/vars.yml` contains only non-sensitive values and `{{ vault_* }}` references — it is safe to commit as-is.
* No plain-text secrets, IPs, usernames, or ports appear anywhere in the repository.
* **`vault_github_runner_registration_token`** is only needed when provisioning or reinstalling a runner — use a short-lived PAT (e.g. 1-day expiry, `repo` scope) and revoke it once provisioning is done.

---

## Build toolchain

Infra owns the toolchain on every host. Pipelines build on the host that runs the service, using exactly what this repository installs, and never install toolchains of their own. One version of each serves every service on a host.

| Toolchain | Source | Pinned in `vars.yml` |
|---|---|---|
| Python | the distribution's `python3.11` and `python3.11-venv` — what piwheels builds against | `python_version` |
| uv | official release binary, checksum-verified | `uv_version` |
| Rust | `rustup`, system-wide in `/opt/rust` — the distribution's `rustc` is too old for edition 2024 | `rustup_version`, `rust_version` |

The toolchain is owned by root and read-only to everyone else; build caches live in the runner account's home.

What a host provides is declared in `/etc/kolumbs/toolchain.env`, which pipelines source:

```sh
TARGET=armv7-unknown-linux-gnueabihf
PYTHON=/usr/bin/python3.11
UV=/usr/local/bin/uv
UV_PYTHON_DOWNLOADS=never       # uv uses PYTHON, never downloads one
RUSTUP_HOME=/opt/rust/rustup
CARGO=/opt/rust/cargo/bin/cargo
```

The target comes from the userland architecture (`dpkg --print-architecture`), not the kernel — a 32-bit Raspberry Pi OS often runs on a 64-bit kernel.

To change a version, change it in `vars.yml` and run the playbook; the sections are tagged, so `--tags rust` (or `python`, `uv`, `manifest`) runs one alone.

---

## Service slots

A service slot is everything a service needs from the host so that its own pipeline can build and deploy it with no further infra changes. `tasks/service_slot.yml` creates one; `main.yml` includes it once per service.

The contract between infra and a service's pipeline:

- the runner account owns `/opt/<service>`; the pipeline puts a release there and points `/opt/<service>/current` at it
- the service runs `/opt/<service>/current/start` as its own `<service>` account (no login), with `PORT` and the service's secrets in its environment
- anything it keeps goes in `$STATE_DIRECTORY` (`/var/lib/<service>`); everything else on the system is read-only to it
- the runner may run exactly `sudo systemctl restart <service>`, and nothing else as root

What a slot provides:

| | |
|---|---|
| `<service>` account | runs the service; no login, no sudo |
| `/opt/<service>` | owned by the runner; releases go here |
| `/etc/<service>/env` | `PORT` and secrets from the vault; root-only, read by systemd before it drops privileges, so neither the service nor the runner can read it |
| `<service>.service` | systemd unit running `current/start`, `Restart=on-failure`, hardened with `ProtectSystem=strict`; enabled, and starts on the first deploy |
| sudoers entry | `systemctl restart <service>` for the runner |
| public route (optional) | a file in `/etc/apache2/kolumbs.net.d/`, proxying a path on the TLS vhost to the service as a websocket |

The TLS vhost itself — certificates and the routes that predate this — is still managed by hand. The playbook adds one `IncludeOptional /etc/apache2/kolumbs.net.d/*.conf` line to it, and every route it owns is a file in that directory. Apache's configuration is tested before every reload.

On first provisioning, the playbook checks that the slot's port is free.

### Current slots

| Service | Port | Route | Vault secrets |
|---|---|---|---|
| botforge | 1609 | `/ws/personal` | `vault_botforge_admin_password` |

### Adding a slot

1. Add its port, route and secret bridges to `group_vars/all/vars.yml`, and the secrets to the vault (`ansible-vault edit group_vars/all/vault.yml`).
2. Include `tasks/service_slot.yml` for it in the service-slot play in `main.yml`, with `slot_name`, `slot_port`, an optional `slot_route` and `slot_env`.
3. Run the playbook; then the service's pipeline can deploy it.

---

## Removing things from managed hosts

Ansible does not automatically remove anything when you delete a task or role from a playbook. If you remove a package, service, or user from the playbook, it will keep running on the host untouched.

To actually remove something, you must explicitly add a task with `state: absent` first, run the playbook, then remove the task:

```yaml
- name: Remove package
  ansible.builtin.apt:
    name: somepackage
    state: absent

- name: Remove user
  ansible.builtin.user:
    name: someuser
    state: absent
```

For roles (e.g. GitHub Actions runner), check the role's documentation for a removal state — most support `state: absent` or similar.

---

## Adding new devices

1. Add the host to `inventory.ini` under the appropriate group (or a new one).
2. Add any new secrets to `group_vars/all/vault.yml` (with `vault_` prefix) and a corresponding bridge entry in `group_vars/all/vars.yml`.
3. Create `host_vars/<hostname>.yml` for device-specific variables (e.g. `ansible_host`).
4. Add a new play to `main.yml` targeting that host/group.
