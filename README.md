# ansible-iroh

[![CI](https://github.com/rayfish/ansible-iroh/actions/workflows/ci.yml/badge.svg)](https://github.com/rayfish/ansible-iroh/actions/workflows/ci.yml)
[![License](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](LICENSE-MIT)
[![Ansible](https://img.shields.io/badge/ansible-2.15%2B-1A1918.svg?logo=ansible&logoColor=white)](https://docs.ansible.com/)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-ansible--iroh-660198.svg)](https://galaxy.ansible.com/)

Ansible roles and playbooks to deploy [`iroh-relay`](https://github.com/n0-computer/iroh/tree/main/iroh-relay) and [`iroh-dns-server`](https://github.com/n0-computer/iroh/tree/main/iroh-dns-server) from the [n0-computer/iroh](https://github.com/n0-computer/iroh) project.

[iroh](https://iroh.computer/) is a modular networking stack that dials cryptographic node IDs instead of IP addresses. The relay helps two peers establish a connection when direct hole-punching fails; the DNS server publishes [pkarr](https://pkarr.org/) records so node IDs can be resolved by human-readable names.

## Features

- Two self-contained roles: `iroh_relay` and `iroh_dns_server`.
- Install from GitHub release tarball (default) or build from source with `cargo install`.
- Hardened systemd units with `CAP_NET_BIND_SERVICE` for binding privileged ports.
- Works on Debian, EL, and SUSE distributions.
- Smoke-test playbook (`tests/render.yml`) for rendering templates locally.
- CI with `ansible-lint`, `yamllint`, syntax checks, and template rendering.

## Table of contents

- [Requirements](#requirements)
- [Supported platforms](#supported-platforms)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Dynamic inventory](#dynamic-inventory)
- [Configuration](#configuration)
- [Production example](#production-example)
- [Service management](#service-management)
- [Firewall](#firewall)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Security](#security)
- [Contributing](#contributing)
- [Releasing](#releasing)
- [License](#license)

## Requirements

- **Control node:** Ansible 2.15+ on Linux or macOS.
- **Target hosts:** A Linux distribution running `systemd`.
- **Access:** `python3` and `sudo`/root access on the target hosts.
- **Source builds only:** A working C toolchain; rustup is installed automatically if `cargo` is missing.

## Supported platforms

| Family | Versions | Notes |
|--------|----------|-------|
| Debian | 11 (Bullseye), 12 (Bookworm), 13 (Trixie) | Tested against the package list in `vars/main.yml`. |
| RedHat | 8, 9, 10 | EL family in `ansible_os_family`. |
| SUSE | openSUSE Leap 15, SLES 15 | Suse family in `ansible_os_family`. |

Architectures: `x86_64` (also reported as `amd64`) and `aarch64` (also `arm64`). Override `iroh_relay_arch_target` / `iroh_dns_arch_target` if your target triple isn't in the map (for example `x86_64-unknown-linux-musl`).

## Installation

Pick one of the following:

### From Ansible Galaxy (recommended)

Install the collection, then reference the roles by their fully qualified
name (`rayfish.iroh.<role>`) from your own playbook:

```bash
ansible-galaxy collection install rayfish.iroh
```

Or pin it in a project's `requirements.yml` (recommended for reproducible runs):

```yaml
# requirements.yml
collections:
  - name: rayfish.iroh
    version: ">=1.0.0"
```

```bash
ansible-galaxy collection install -r requirements.yml
```

```yaml
# playbook.yml
- hosts: relays
  roles:
    - rayfish.iroh.iroh_relay

- hosts: dns
  roles:
    - rayfish.iroh.iroh_dns_server
```

```bash
ansible-playbook -i inventory.ini playbook.yml
```

### As a git checkout (development)

```bash
git clone https://github.com/rayfish/ansible-iroh.git
cd ansible-iroh
ansible-playbook playbooks/site.yml
```

The bundled `playbooks/site.yml`, `deploy_relay.yml`, and `deploy_dns.yml`
reference the roles by short name and rely on `ansible.cfg` (`roles_path = ./roles`),
so they run directly from a checkout.

## Quick start

1. **Edit `inventory/hosts.ini`** to point at your hosts:

   ```ini
   [iroh_relay]
   relay.example.com ansible_host=192.0.2.10

   [iroh_dns]
   dns.example.com ansible_host=192.0.2.11

   [iroh:children]
   iroh_relay
   iroh_dns

   [iroh:vars]
   ansible_user=root
   ```

2. **Edit `group_vars/all.yml`** (or create per-host `host_vars/<host>.yml` files) for your domain names, certificates, and listener ports. The shipped defaults bind unprivileged ports (`3340` for relay, `5300`/`8080` for DNS) so a first run works without root-bound ports.

3. **Run the playbooks:**

   ```bash
   # Deploy both services based on inventory groups
   ansible-playbook playbooks/site.yml

   # Deploy only the relay
   ansible-playbook playbooks/deploy_relay.yml

   # Deploy only the DNS server
   ansible-playbook playbooks/deploy_dns.yml
   ```

   If your remote user requires a sudo password, add `-K` to the command.

   > `host_key_checking` is enabled by default in `ansible.cfg`. If your target
   > hosts are not yet in `~/.ssh/known_hosts`, either SSH to them once manually
   > or run `ssh-keyscan -H <host> >> ~/.ssh/known_hosts` first.

## Dynamic inventory

The playbooks don't depend on the static `inventory/hosts.ini` — they target the
groups **`iroh_relay`** and **`iroh_dns`**. That group membership is the only
contract, so any inventory source works: a cloud plugin (`amazon.aws.aws_ec2`,
`google.cloud.gcp_compute`, …), an inventory script, or the static file.

`ansible.cfg` points `inventory` at the whole `inventory/` directory, so Ansible
loads and **merges every source** in it. To use a dynamic source, just drop its
config into `inventory/` alongside `hosts.ini` — no `-i` flag needed:

```text
inventory/
  hosts.ini                  # static example (delete or keep)
  aws_ec2.yml                # your cloud plugin config
  zz_constructed.yml         # optional: map host vars -> iroh_* groups
```

The included `inventory/zz_constructed.yml.example` is a provider-agnostic
template. Give each host an `iroh_role` of `relay` or `dns` (via a cloud tag,
`host_vars/`, or `group_vars/`) and the `constructed` plugin places it in the
right group:

```yaml
# inventory/zz_constructed.yml  (rename from the .example file)
plugin: ansible.builtin.constructed
strict: false
keyed_groups:
  - key: iroh_role        # iroh_role=relay -> group "iroh_relay"
    prefix: iroh          # iroh_role=dns   -> group "iroh_dns"
    separator: "_"
```

> The `zz_` prefix matters: a directory inventory is parsed alphabetically, and
> `constructed` can only group hosts defined by sources parsed **before** it.
> The prefix makes it sort last. Verify your groups with
> `ansible-inventory --graph` before deploying.

A cloud plugin can populate the groups directly instead, e.g. with `amazon.aws.aws_ec2`:

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
groups:
  iroh_relay: "'relay' in (tags.iroh_role | default(''))"
  iroh_dns:   "'dns' in (tags.iroh_role | default(''))"
```

## Configuration

All tunables live in `group_vars/all.yml` and each role's `defaults/main.yml`. Override per-host in `host_vars/` or per-group in `group_vars/` as needed. The values shown below are the defaults; the role's own `defaults/main.yml` is the source of truth.

### iroh-relay variables

| Variable | Default | Description |
|----------|---------|-------------|
| `iroh_relay_version` | `"1.0.0"` | Release version to install. |
| `iroh_relay_install_method` | `"release"` | `"release"` downloads the GitHub release tarball; `"source"` builds from git. |
| `iroh_relay_arch_target` | `""` | Override the target triple (e.g. `x86_64-unknown-linux-musl`). Affects release installs only. |
| `iroh_relay_http_bind_addr` | `"[::]:3340"` | HTTP listener address. Use `[::]:80` or `0.0.0.0:443` in production. |
| `iroh_relay_enable_relay` | `true` | Enable relay traffic. |
| `iroh_relay_enable_metrics` | `true` | Enable the metrics server. |
| `iroh_relay_metrics_bind_addr` | `""` | Optional metrics bind address (e.g. `[::]:9090`). Empty means the upstream default. |
| `iroh_relay_enable_quic_addr_discovery` | `false` | Enable QUIC address discovery. Requires `iroh_relay_tls`. |
| `iroh_relay_access` | `"everyone"` | Simple access mode. Must be `"everyone"` or `""`; complex access goes in `iroh_relay_extra_config`. |
| `iroh_relay_tls` | `""` | Raw TOML block for the `[tls]` section. Required when QUIC discovery is enabled. |
| `iroh_relay_extra_config` | `""` | Raw TOML appended at the end of the config. Use for complex `access.*` blocks. |
| `iroh_relay_dev_mode` | `false` | Add `--dev` to the CLI. Local testing only. |
| `iroh_relay_extra_args` | `""` | Extra arguments passed to the systemd unit's `ExecStart`. |
| `iroh_relay_source_repo` | `https://github.com/n0-computer/iroh.git` | Git URL for source installs. |
| `iroh_relay_source_version` | `v{{ iroh_relay_version }}` | Git ref to check out. |
| `iroh_relay_source_dir` | `/usr/local/src/iroh-relay` | Clone destination for source installs. |

### iroh-dns-server variables

| Variable | Default | Description |
|----------|---------|-------------|
| `iroh_dns_version` | `"1.0.0"` | Release version to install. |
| `iroh_dns_install_method` | `"release"` | `"release"` or `"source"`. |
| `iroh_dns_arch_target` | `""` | Override the target triple. Affects release installs only. |
| `iroh_dns_port` | `5300` | DNS listener port. Use `53` in production. |
| `iroh_dns_bind_addr` | `"0.0.0.0"` | DNS listener bind address. |
| `iroh_dns_default_soa` | `"dns1.irohdns.example.org hostmaster…"` | SOA record for the zone. |
| `iroh_dns_default_ttl` | `300` | Default TTL for synthesized records. |
| `iroh_dns_origins` | `["irohdns.example.org", "."]` | DNS origins for published pkarr zones. |
| `iroh_dns_rr_a` | `""` | Optional apex A record (e.g. `"192.0.2.10"`). |
| `iroh_dns_rr_ns` | `""` | Optional apex NS record (e.g. `"ns1.irohdns.example.org."`). |
| `iroh_dns_http_enabled` | `true` | Enable the HTTP API listener. |
| `iroh_dns_http_port` | `8080` | HTTP API port. |
| `iroh_dns_http_bind_addr` | `"0.0.0.0"` | HTTP API bind address. |
| `iroh_dns_https_enabled` | `false` | Enable the HTTPS / DNS-over-HTTPS listener. |
| `iroh_dns_https_port` | `8443` | HTTPS port. Use `443` in production. |
| `iroh_dns_https_bind_addr` | `"0.0.0.0"` | HTTPS bind address. |
| `iroh_dns_https_domains` | `["irohdns.example.org"]` | Domains the certificate is valid for. |
| `iroh_dns_https_cert_mode` | `"lets_encrypt"` | `self_signed`, `manual`, or `lets_encrypt`. |
| `iroh_dns_https_letsencrypt_contact` | `"hostmaster@example.org"` | Required for `lets_encrypt`. |
| `iroh_dns_https_letsencrypt_prod` | `true` | Use the Let's Encrypt production API. |
| `iroh_dns_metrics_disabled` | `false` | Disable the metrics endpoint entirely. |
| `iroh_dns_metrics_bind_addr` | `"127.0.0.1:9117"` | Metrics bind address. |
| `iroh_dns_mainline_enabled` | `false` | Enable mainline DHT fallback. |
| `iroh_dns_mainline_bootstrap` | `[]` | Mainline DHT bootstrap nodes (list of strings). |
| `iroh_dns_extra_config` | `""` | Raw TOML appended at the end of the config. |
| `iroh_dns_source_repo` | `https://github.com/n0-computer/iroh.git` | Git URL for source installs. |
| `iroh_dns_source_version` | `v{{ iroh_dns_version }}` | Git ref to check out. |
| `iroh_dns_source_dir` | `/usr/local/src/iroh-dns-server` | Clone destination for source installs. |

## Production example

```yaml
---
# group_vars/all.yml

# iroh-relay on 443 with QUIC address discovery
iroh_relay_http_bind_addr: "0.0.0.0:443"
iroh_relay_enable_quic_addr_discovery: true
iroh_relay_tls: |
  [tls]
  cert_mode = "Manual"
  manual_cert_path = "/etc/letsencrypt/live/relay.example.org/fullchain.pem"
  manual_key_path  = "/etc/letsencrypt/live/relay.example.org/privkey.pem"

# iroh-dns-server on 53 / 443 with Let's Encrypt
iroh_dns_port: 53
iroh_dns_origins: ["irohdns.example.org", "."]
iroh_dns_default_soa: >-
  dns1.irohdns.example.org hostmaster.irohdns.example.org
  0 10800 3600 604800 3600

iroh_dns_https_enabled: true
iroh_dns_https_port: 443
iroh_dns_https_domains: ["irohdns.example.org"]
iroh_dns_https_cert_mode: "lets_encrypt"
iroh_dns_https_letsencrypt_contact: "hostmaster@example.org"
iroh_dns_https_letsencrypt_prod: true
```

> The systemd units include `AmbientCapabilities=CAP_NET_BIND_SERVICE`, so the services can bind to privileged ports without running as root. Make sure the service user can read certificate files.

### Important notes

#### iroh-relay

- The default HTTP bind address is `[::]:3340` so the service does not need root privileges.
- To bind `80`/`443`, set `iroh_relay_http_bind_addr` accordingly; `CAP_NET_BIND_SERVICE` is already granted.
- If you enable `iroh_relay_enable_quic_addr_discovery`, a `[tls]` section **must** be supplied. `cert_mode = "LetsEncrypt"` also requires at least one entry in `iroh_relay_tls` `hostname = ["…"]`.
- The string form of `iroh_relay_access` only accepts `"everyone"`. Allowlist, denylist, shared tokens, and HTTP callout go in `iroh_relay_extra_config`. Set `iroh_relay_access: ""` to avoid a duplicate `access` key.

#### iroh-dns-server

- Default listeners use unprivileged ports (`5300` for DNS, `8080` for HTTP) so the role works out of the box.
- Set `iroh_dns_port: 53` and `iroh_dns_https_port: 443` for production.
- `cert_mode` accepts `self_signed`, `manual`, or `lets_encrypt` per the upstream binary's config schema.
- State is stored under `/var/lib/iroh-dns`; the signed-packet database and ACME cache are created there automatically.

### Switching install methods

The role picks between `install_release.yml` and `install_source.yml` based on `iroh_relay_install_method` / `iroh_dns_install_method`. To switch an existing host from one method to the other:

1. Set the variable to the new method.
2. Manually remove the old binary if needed (e.g. `rm /usr/local/bin/iroh-relay`).
3. Re-run the playbook; the new method will install the binary in place.

Source builds are slow (several minutes) and only useful when you need a version newer than the latest GitHub release or want to apply local patches.

## Service management

```bash
systemctl status iroh-relay
systemctl status iroh-dns-server

journalctl -u iroh-relay -f
journalctl -u iroh-dns-server -f

systemctl restart iroh-relay
systemctl restart iroh-dns-server
```

Service units are managed by Ansible; manual edits will be overwritten on the next run. To customize, override the corresponding template variable.

## Firewall

The roles do not manage the host firewall. Make sure your firewall allows the ports you configure, for example:

| Service | Ports |
|---------|-------|
| iroh-relay | `80/tcp`, `443/tcp`, `7842/udp` (QUIC address discovery), `9090/tcp` (metrics) |
| iroh-dns-server | `53/tcp`, `53/udp`, `80/tcp`, `443/tcp`, `9117/tcp` (metrics) |

The QUIC port is `7842/udp` (upstream default); iroh-relay binds `https_bind_addr + 1` from the TLS configuration when QUIC discovery is enabled.

## Testing

CI runs on every push and pull request (`.github/workflows/ci.yml`) and includes:

- `ansible-lint` against the whole project.
- `yamllint` for YAML formatting.
- `ansible-playbook --syntax-check` on every playbook.
- `tests/render.yml` to render the templates end-to-end on `localhost`.

Run the same checks locally:

```bash
pip install ansible-core ansible-lint yamllint
ansible-lint
yamllint .
ansible-playbook --syntax-check playbooks/site.yml
ansible-playbook --syntax-check playbooks/deploy_relay.yml
ansible-playbook --syntax-check playbooks/deploy_dns.yml
ansible-playbook tests/render.yml
```

The `tests/render.yml` playbook writes `/tmp/iroh-relay-config.toml` and `/tmp/iroh-dns-config.toml` so you can inspect the generated TOML without touching a remote host.

## Troubleshooting

### `cargo install --path iroh-relay: path does not exist`

The source installer requires the path to start with `./`. Make sure you're on the latest version of these roles — the bug was fixed before the 1.0.0 release. If you're pinning an older tag, switch to `release` install or apply the `--path ./iroh-relay` patch locally.

### Service fails with `Address already in use`

Another process is bound to the port. Check with `ss -ltnp 'sport = :443'` and either stop the conflicting service or move iroh to a different port via `iroh_relay_http_bind_addr` / `iroh_dns_port`.

### Service fails with `permission denied` on a privileged port

The systemd unit grants `CAP_NET_BIND_SERVICE`, but if you've overridden `AmbientCapabilities` you may have removed it. Don't override the security-hardening directives unless you know what you're doing.

### `LetsEncrypt needs at least one hostname`

When `iroh_relay_enable_quic_addr_discovery: true` with `cert_mode = "LetsEncrypt"`, the TLS block must include at least one hostname, e.g.:

```yaml
iroh_relay_tls: |
  [tls]
  cert_mode = "LetsEncrypt"
  hostname = ["relay.example.org"]
  contact = "hostmaster@example.org"
```

### Templates render but the service won't start

Run `journalctl -u iroh-relay -e` or `journalctl -u iroh-dns-server -e` for the most recent error. Common culprits: a typo in the TOML (render the template via `tests/render.yml` and validate with `tomlq`), or an unreadable certificate file (check `ls -l` and the service user's group membership).

## Security

Both roles ship with systemd hardening enabled by default:

- `NoNewPrivileges=true`
- `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`
- `ReadWritePaths` limited to config and data directories
- `ReadOnlyPaths=-/etc/letsencrypt` so certificates are readable
- `ProtectKernelTunables`, `ProtectKernelModules`, `ProtectControlGroups`
- `AmbientCapabilities=CAP_NET_BIND_SERVICE` only

If you report a security issue, please follow responsible disclosure and contact the maintainers before opening a public issue.

## Contributing

Issues and pull requests are welcome. Before opening a PR:

1. Run `ansible-lint`, `yamllint`, and `ansible-playbook tests/render.yml` locally — all three must pass.
2. Keep variable additions documented in both the role's `defaults/main.yml` and the README configuration tables.
3. Bump `CHANGELOG.md` under `[Unreleased]` with a short description of the change.
4. Don't introduce a dependency on Galaxy or the network from inside a role — keep roles self-contained.

Roles follow the standard Ansible role directory layout ([reference](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)).

## Releasing

This repository is packaged as the `rayfish.iroh` Ansible Galaxy collection
(`galaxy.yml`). To cut a release:

1. Bump `version` in `galaxy.yml` and move the `[Unreleased]` entries in
   `CHANGELOG.md` under the new version.
2. Build the artifact (CI also does this on every push):

   ```bash
   ansible-galaxy collection build --output-path dist/
   ```

3. Publish to Galaxy with your API token (from <https://galaxy.ansible.com/me/preferences>):

   ```bash
   ansible-galaxy collection publish dist/rayfish-iroh-<version>.tar.gz --token <your-galaxy-token>
   ```

4. Tag the release: `git tag v<version> && git push --tags`.

The `build_ignore` list in `galaxy.yml` keeps development-only files
(`inventory/`, `group_vars/`, `tests/`, `ansible.cfg`, CI config) out of the
published artifact.

## License

Dual-licensed under [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at your option. Same as the upstream iroh project.
