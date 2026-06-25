# Security Policy

## Supported versions

This project tracks the latest release. Security fixes are applied on top of the
most recent tagged version.

## Reporting a vulnerability

Please report security issues privately rather than opening a public issue.

- Preferred: open a [GitHub security advisory](https://github.com/rayfish/ansible-iroh/security/advisories/new).
- Alternatively, email **dario@rayfish.xyz**.

Please include enough detail to reproduce the issue. You can expect an initial
response within a few days. We follow responsible disclosure and will coordinate
on a fix and disclosure timeline before any public announcement.

## Scope

This repository deploys upstream [`iroh`](https://github.com/n0-computer/iroh)
binaries. Vulnerabilities in `iroh-relay` or `iroh-dns-server` themselves should
be reported to the [upstream project](https://github.com/n0-computer/iroh).
Issues in the Ansible roles, templates, or systemd hardening belong here.
