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
