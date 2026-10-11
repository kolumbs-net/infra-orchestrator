# TODO

## Ansible should own the TLS vhost

**Today:** the 443 vhost (`kolumbs-ssl.conf`) is hand-written on the host. It holds
the certificate paths, `DocumentRoot`, and the routes that predate Ansible. A
route Ansible manages has to sit inside that vhost — at server level it would
apply to every vhost, and a second `*:443` vhost with the same `ServerName` is
ignored. So the playbook adds one `IncludeOptional` line to the hand-written file
(`apache_tls_vhost`) and owns a directory of route files (`apache_routes_dir`),
one per service slot.

**Problems with that:**

- Ansible edits a file it does not own. If the vhost is rewritten by hand and the
  include line is lost, every Ansible-managed route silently stops working until
  the next playbook run.
- The host cannot be rebuilt from this repository: the vhost, its certificates
  and the older routes exist only on the machine.
- `apache_tls_vhost` and `apache_routes_dir` are host-specific but live in
  `group_vars/all`; they belong in `host_vars` (or the `web_server` group).
- The routes directory is named after the domain; a neutral name would do.

**Better:** render the whole TLS vhost from a template, with routes as data —
backends from the vault, like everything else that describes the machine:

```yaml
apache_tls_routes:
  - { path: "{{ vault_<service>_route }}", backend: "ws://localhost:{{ vault_<service>_port }}/" }
```

A service slot adds its own entry; the include line, the drop-in directory and
both variables go away.

**How:**

1. Write the template so that it reproduces today's file, including the existing
   routes, `DocumentRoot` and `ProxyPreserveHost`.
2. Run `ansible-playbook main.yml --limit pi-server-1 --check --diff` and switch
   only once the diff is empty or fully understood.
3. Then certificates: make their paths variables at least, and ideally let
   Ansible manage issuance and renewal (e.g. certbot) instead of placing them by
   hand.

The older port-based vhosts (one TLS port per service) either get the same
treatment or disappear as each service moves to a path on 443, which also
avoids firewalls that block uncommon ports.

## Move pi-server-1 to a 64-bit OS

**Today:** the Pi runs a 32-bit userland (`armhf`). Node.js 24 has no 32-bit ARM
build, and from runner 2.338.0 GitHub's runner needs Node 24 for its own service
wrapper, so the runner is pinned to 2.337.0 with self-update disabled
(`github_runner_version`). That only buys time: GitHub stops accepting runners
that fall too far behind, and JavaScript actions such as `actions/checkout` are
moving to Node 24 as well. Until then, workflows on this runner should use plain
`run:` steps rather than JavaScript actions.

**Better:** reinstall with 64-bit Raspberry Pi OS (`arm64`). The runner, Node 24
and all actions work; every compiled Python package has prebuilt `aarch64` wheels
on PyPI, so piwheels is no longer needed. The playbook already maps `arm64` for
both the toolchain and the runner, so then remove the pin and
`--disableupdate`.

**Before that:** whatever runs on the Pi but is not in this repository has to be
restorable — the hand-managed vhosts and certificates (see above) and the
services that predate the service slots. New devices, such as the GSM Pi, should
start on 64-bit.


## Account pool for botforge workers

botforge v1 gives every user a private assistant whose tools (arbitrary Python)
run in a worker process of their own, as their own Linux account, so the
operating system keeps users out of each other's files and processes. See
"v1 multi-user" in botforge's TODO.md for the design.

Accounts cannot be created at runtime without root, so infra provides a pool:

- Accounts `botforge-u001` … `botforge-uNNN`, no login, all in a group
  `botforge-workers`. The pool size is a variable; growing it means changing the
  number and re-running the playbook.
- A folder per account, `/var/lib/botforge/users/uNNN/`, owned by that account,
  mode `0700`. The `botforge` router never needs to read inside it.
- One sudoers rule, the only escalation the botforge service gets:
  `botforge ALL=(%botforge-workers) NOPASSWD: /opt/botforge/current/worker`.
- `NoNewPrivileges=false` for the botforge service unit, which currently blocks
  every sudo call. Ideally this is a per-slot option, not a change for all slots.

Generic enough to become a slot option ("this slot runs per-user workers"), so
another multi-user service could reuse it.
