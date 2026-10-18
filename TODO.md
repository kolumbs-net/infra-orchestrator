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


## Standard ports for service slots

**Today:** a slot has exactly one port: `slot_port`, an optional
`slot_protocol` (`tcp` or `udp`) and an optional `slot_route`, given to the
service as `PORT`. That fit botforge, but services need more: the telemetry
collector receives probe readings over UDP on the LAN and should also serve a
public, read-only HTTP API on a TCP port of its own, behind an Apache route.
Every future service will need some set of ports too.

**Problems:**

- One port per slot. Adding a second means a one-off parameter per case, with
  names that only fit that one service.
- The port-free check runs only when a slot is first created, so a port added
  to an existing slot is never checked.
- Routes always proxy as websockets (`ws://`), so a plain HTTP service can't
  have one.
- The machine's convention, one port range for internal (LAN-only) services
  and another for public ones, lives only in people's heads. Nothing stops a
  public port from being taken from the internal range, or the other way round.

**Better:** every slot declares its ports in one list, `slot_ports`:

| Field | Meaning |
|---|---|
| `name` | lowercase; the service gets it as `<NAME>_PORT` (`api` → `API_PORT`) |
| `port` | the number, from the vault |
| `protocol` | `tcp` or `udp` |
| `exposure` | `internal` (LAN only) or `external` (public) |
| `route` | optional, external TCP ports only: public path on the TLS vhost |
| `route_scheme` | `ws` (default) or `http` |

- The unit gets one `Environment=<NAME>_PORT=<port>` per entry; `PORT`,
  `slot_port`, `slot_protocol` and `slot_route` go away.
- The play checks each entry before changing anything: a valid and unique
  name, a unique port, tcp or udp, and a `route` only on an external TCP port.
  **The exposure must match the port's range.** The internal and external
  ranges come from the vault (e.g. `vault_internal_port_range`,
  `vault_external_port_range`), like every other detail of the machine, so the
  convention is enforced without being written into the repository.
- Each port is checked as free, with `ss -lt` or `ss -lu` for its protocol,
  whenever the current unit doesn't already declare it. A port added to an
  existing slot is then checked too; ports the running service holds are
  skipped.
- The route file stays one per slot (`<slot>.conf`), with one `ProxyPass` per
  routed port: `ws://localhost:<port>/` for `ws`, and `http://localhost:<port>/`
  plus `ProxyPassReverse` for `http`. `proxy_http` is already enabled.

**Slots after the change:**

- botforge: one port, `websocket` (tcp, external, its current route, `ws`).
  Its route file renders the same line as today.
- telemetry: `collector` (udp, internal, today's port) and `api` (tcp,
  external, new, with an `http` route). New vault entries:
  `vault_telemetry_api_port` and `vault_telemetry_route`, each with an assert
  and a bridge in `group_vars/all/vars.yml`.

**How, in this order:**

1. botforge reads its port as `${WEBSOCKET_PORT:-$PORT}` (see botforge's
   TODO), deployed first, so it keeps running through the switch.
2. This change, plus the new vault entries and the port ranges. Check with a
   deliberately wrong port (an internal port taken from the external range)
   that the play fails at the checks, then `--check --diff`: botforge's unit
   should change only `PORT` → `WEBSOCKET_PORT`, and its route file not at all.
3. The telemetry collector reads `COLLECTOR_PORT` and `API_PORT` (see
   telemetry's TODO). Between steps 2 and 3 it stops, because it still expects
   `PORT`; a short gap in readings is acceptable there.
4. Update the README: the slot contract (`<NAME>_PORT` per declared port), the
   fields and the range check under "Adding a slot", and each slot's ports
   with their exposure in the slot table.
