# `openvpn` role

Deploys an OpenVPN server on Debian and Ubuntu from a PKI you hand it: the CA
certificate, the server certificate and key, a tls-crypt key and, optionally,
a CRL — each as content (e.g. from Ansible Vault) or as a file on the
controller. It runs the server as the distribution's
`openvpn-server@<instance>` unit and can enable IP forwarding and NAT for the
clients.

What it deliberately leaves to other tools: **creating keys and certificates**
(easy-rsa, step-ca, Vault, …), client profiles, and opening the VPN port or
forwarding in the host firewall (e.g. the `homelab.linux` role's UFW rules).

## Requirements

* ansible-core ≥ 2.15
* `ansible.posix` (also listed in [`requirements.yml`](requirements.yml)).
* A Debian (bookworm, trixie) or Ubuntu (jammy, noble) host with systemd. The
  role installs `openvpn` itself, and `nftables` when NAT is enabled.
* An existing PKI: CA certificate, server certificate with *TLS Web Server
  Authentication*, its **unencrypted** key, and a tls-crypt key
  (`openvpn --genkey secret tls-crypt.key`). Client certificates need *TLS Web
  Client Authentication* — the server checks for it (`remote-cert-tls client`).
* `become: true` — the role installs packages, writes to `/etc` and manages a
  systemd unit.

## What the role does

1. **Validates input** on the controller before touching the host: the
   platform, the instance name, the listener and network settings (CIDR
   notation, a client pool of /29 or larger without host bits), and every PKI
   file — given exactly once (content *or* `_src`) and containing the right PEM
   block, so a swapped file or an encrypted key fails the play instead of the
   service. Types and choices are checked by
   [`meta/argument_specs.yml`](meta/argument_specs.yml).
2. **Installs** `openvpn` (and `nftables` for NAT).
3. **Installs the PKI** into `/etc/openvpn/server/<instance>/`: keys `0600`,
   certificates and CRL `0644` (OpenVPN reads the CRL after dropping root).
4. **Renders** `/etc/openvpn/server/<instance>.conf`.
5. **Enables IP forwarding** and, if asked, **NAT** — an nftables table that a
   systemd drop-in loads when the server starts and deletes when it stops.
6. **Enables and starts** `openvpn-server@<instance>`, restarting it when its
   configuration, certificates or keys change.

On the managed host it leaves (for the default instance `server`):

```text
/etc/openvpn/server/
├── server.conf                      # the server configuration
└── server/
    ├── ca.crt                       # 0644
    ├── server.crt                   # 0644
    ├── server.key                   # 0600
    ├── tls-crypt.key                # 0600
    ├── crl.pem                      # 0644, only with a CRL
    └── nat.nft                      # only with NAT
/etc/systemd/system/openvpn-server@server.service.d/
└── nat.conf                         # only with NAT
/etc/sysctl.d/99-openvpn-server.conf # only with IP forwarding
```

## Variables

See [`defaults/main.yml`](defaults/main.yml) for the full commented list.

**PKI** — give each file *either* as content *or* as a path on the
controller (`…_src`, looked up like any `file` lookup), never both:

| Content variable        | Path variable               | File            | Required |
|-------------------------|-----------------------------|-----------------|----------|
| `openvpn_ca_cert`       | `openvpn_ca_cert_src`       | `ca.crt`        | yes      |
| `openvpn_server_cert`   | `openvpn_server_cert_src`   | `server.crt`    | yes      |
| `openvpn_server_key`    | `openvpn_server_key_src`    | `server.key`    | yes      |
| `openvpn_tls_crypt_key` | `openvpn_tls_crypt_key_src` | `tls-crypt.key` | yes      |
| `openvpn_crl`           | `openvpn_crl_src`           | `crl.pem`       | no       |

**Server:**

| Variable                     | Default                              | Purpose |
|------------------------------|--------------------------------------|---------|
| `openvpn_state`              | `present`                            | `present` / `absent` |
| `openvpn_instance`           | `server`                             | Instance name: config file, PKI directory, unit |
| `openvpn_port`               | `1194`                               | Listening port |
| `openvpn_proto`              | `udp`                                | `udp` / `tcp` / `udp6` / `tcp6` |
| `openvpn_local`              | `""`                                 | Bind address, empty = all |
| `openvpn_dev`                | `tun`                                | Tunnel device |
| `openvpn_server_network`     | `10.8.0.0/24`                        | Client address pool (CIDR) |
| `openvpn_client_to_client`   | `false`                              | Let clients reach each other |
| `openvpn_keepalive_interval` | `10`                                 | Ping interval (s) |
| `openvpn_keepalive_timeout`  | `120`                                | Peer timeout (s) |
| `openvpn_push_routes`        | `[]`                                 | Networks (CIDR) routed through the VPN |
| `openvpn_redirect_gateway`   | `false`                              | Route all client traffic through the VPN |
| `openvpn_dns_servers`        | `[]`                                 | DNS servers pushed to clients |
| `openvpn_dns_domain`         | `""`                                 | DNS search domain pushed to clients |
| `openvpn_data_ciphers`       | `[AES-256-GCM, AES-128-GCM, CHACHA20-POLY1305]` | Data channel ciphers |
| `openvpn_tls_version_min`    | `"1.2"`                              | Oldest TLS version accepted |
| `openvpn_user` / `_group`    | `nobody` / `nogroup`                 | Unprivileged identity after start-up |
| `openvpn_verb`               | `3`                                  | Log verbosity |
| `openvpn_extra_config`       | `[]`                                 | Extra directives, one per line |
| `openvpn_ip_forward`         | `true`                               | Enable IPv4 forwarding |
| `openvpn_nat_masquerade`     | `false`                              | Masquerade client traffic |
| `openvpn_nat_interface`      | default-route interface              | Outgoing interface for NAT |
| `openvpn_no_log`             | `true`                               | Hide key contents from the output |

## Usage

```yaml
# group_vars/vpn.yml
openvpn_ca_cert_src: pki/ca.crt                   # files on the controller ...
openvpn_server_cert_src: pki/issued/vpn.crt
openvpn_crl_src: pki/crl.pem
openvpn_server_key: "{{ vault_openvpn_server_key }}"         # ... or content
openvpn_tls_crypt_key: "{{ vault_openvpn_tls_crypt_key }}"

openvpn_push_routes:
  - 192.168.1.0/24
openvpn_dns_servers:
  - 192.168.1.1
openvpn_nat_masquerade: true
```

```yaml
- hosts: vpn
  become: true
  roles:
    - role: homelab.openvpn
```

Several servers on one host (e.g. UDP and TCP) are separate instances — run
the role once per `openvpn_instance`, each with its own port and network.

### Rotating the PKI

Change the variables and re-run, or only that part with `--tags openvpn_pki`.
New certificates or keys restart the server. A new **CRL does not** —
OpenVPN re-reads it for every new connection and renegotiation (hourly by
default), so connected clients stay up and a revoked one is cut off at its
next renegotiation at the latest.

### Removing

```bash
ansible-playbook playbook.yml -e openvpn_state=absent
```

Stops and disables `openvpn-server@<instance>`, and removes its
configuration, PKI directory, drop-in directory (with the NAT rules) and
sysctl file. The `openvpn` package stays, as do other instances. The running
`ip_forward` value is left alone until the next reboot.

## Tags

| Tag               | Tasks |
|-------------------|-------|
| `always`          | Input validation |
| `openvpn`         | Everything below |
| `openvpn_install` | Package installation |
| `openvpn_pki`     | Certificates, keys, CRL |
| `openvpn_config`  | Server configuration |
| `openvpn_routing` | IP forwarding and NAT |
| `openvpn_service` | Enabling and starting the server |

## Notes

* **No PKI generation, by design.** The role only copies what it is given and
  checks it looks right. Whether the server key matches its certificate shows
  when the service starts — the role then fails, with the reason in
  `journalctl -u openvpn-server@<instance>`.
* **`dh none`**: key exchange uses ECDHE only, so there are no DH parameters
  to generate or pass in. Works with RSA and EC certificates alike.
* **Firewall**: the role does not open `openvpn_port` and does not touch the
  FORWARD chain. Without NAT, the LAN needs a route back to
  `openvpn_server_network` via the VPN host. With NAT, a FORWARD policy that
  drops (UFW's `deny` routed default, Docker) still blocks the clients — allow
  the tunnel traffic there.
* **NAT lives with the service.** Its nftables table exists exactly while the
  server runs, in its own table next to whatever else manages the firewall
  (UFW, Docker, iptables-nft). Switching NAT off deletes the table without
  restarting the server.
* **The configuration must live in `/etc/openvpn/server/`**: the
  distribution's unit runs `openvpn --config <instance>.conf` from there.
* `--diff` never shows key contents (`openvpn_no_log`, `diff: false`). Set
  `openvpn_no_log: false` only to debug a failing key task.

## Testing

A Molecule scenario runs the role in Debian 13 and Ubuntu 24.04 containers.
[`prepare.yml`](molecule/default/prepare.yml) creates a throwaway PKI on the
controller (nothing secret is committed), the role gets it in both input
forms, and after the idempotence check
[`verify.yml`](molecule/default/verify.yml) asserts that the server is
running, listening and holding its keys `0600`, that forwarding and NAT are in
place, that a client connects and receives the pushed routes and DNS, and
that a client with a revoked certificate is turned away. CI runs it, together
with both linters, on every pull request:

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt     # same pinned versions as CI
yamllint --strict .
ansible-lint
molecule test
```

ansible-lint and Molecule install the collections from
[`requirements.yml`](requirements.yml) on their own.

## Releases

PRs are squash-merged, so the PR title becomes the commit on `main`. It must be
a [Conventional Commit](https://www.conventionalcommits.org/) (a check enforces
this), because it decides the next version and the changelog entry:

| PR title                                              | Next release | Changelog       |
|-------------------------------------------------------|--------------|-----------------|
| `feat!: …` or a `BREAKING CHANGE:` footer             | major        | ⚠ Breaking      |
| `feat: …`                                             | minor        | Features        |
| `fix:` / `perf:` / `revert:` / `docs: …`              | patch        | own section     |
| `ci:` / `test:` / `refactor:` / `build:` / `style:` / `chore: …` | none | not listed |

After each merge, [release-please](https://github.com/googleapis/release-please)
opens or updates a release PR with the version bump and the new `CHANGELOG.md`
entry. Merging that PR tags the release (e.g. `1.2.0`) and publishes a GitHub
release.

PRs opened with the default `GITHUB_TOKEN` do not trigger workflows, so the
release PR gets no CI checks. If branch protection requires them, add a
fine-grained PAT as the `RELEASE_PLEASE_TOKEN` secret.

## License

MIT
