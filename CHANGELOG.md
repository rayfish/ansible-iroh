# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Ansible Galaxy collection packaging (`galaxy.yml`) under the `rayfish.iroh`
  namespace, so the roles install via `ansible-galaxy collection install`.
- CI job that builds the collection artifact on every push.
- `SECURITY.md` with a vulnerability-reporting contact.

## [1.0.0] - 2026-06-25

### Added
- Initial public release.
- Roles `iroh_relay` and `iroh_dns_server`.
- Release (GitHub tarball) and source (`cargo install`) install methods.
- Systemd units with security hardening and `CAP_NET_BIND_SERVICE`.
- `meta/main.yml` for both roles (Ansible Galaxy metadata).
- Dual MIT / Apache-2.0 `LICENSE-MIT` and `LICENSE-APACHE` files.
- `.ansible-lint`, `.yamllint`, and `.gitignore`.
- GitHub Actions CI (`.github/workflows/ci.yml`) running `ansible-lint`,
  `yamllint`, syntax-check, and the `tests/render.yml` template smoke test.
- Local template-rendering smoke test (`tests/render.yml`).
- Validation of `iroh_relay_install_method` / `iroh_dns_install_method`
  (`release` or `source`), `iroh_relay_access` (`everyone` or `""`),
  `iroh_relay_http_bind_addr` (non-empty), and `iroh_dns_https_cert_mode`
  (when HTTPS is enabled) — a typo fails fast instead of silently skipping work.
- Source install fails fast with a clear error on unsupported OS families.
