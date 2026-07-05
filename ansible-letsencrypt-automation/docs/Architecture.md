# Architecture

## Overview

This project automates end-to-end SSL certificate lifecycle management for
NGINX-fronted web servers using Certbot and Let's Encrypt.

```
                       ┌─────────────────────────┐
                       │   Control Node (Ansible)│
                       └────────────┬────────────┘
                                    │ SSH
                                    ▼
                 ┌────────────────────────────────────┐
                 │            webservers group          │
                 │  ┌───────────────┐ ┌───────────────┐ │
                 │  │   web01        │ │   web02       │ │
                 │  │  NGINX + certs │ │  NGINX + certs│ │
                 │  └───────────────┘ └───────────────┘ │
                 └────────────────────────────────────┘
                                    │
                                    ▼
                       ┌─────────────────────────┐
                       │   Let's Encrypt ACME CA │
                       └─────────────────────────┘
```

## Execution Flow

1. **Install** - `certbot`, `python3-certbot-nginx`, and `python3-certbot-dns-cloudflare` are installed via `apt`.
2. **Issue** - For each domain in `certbot_domains`, `certbot certonly --nginx` requests a certificate. The `creates` guard makes this idempotent - already-issued certs are skipped.
3. **Harden** - A Jinja2-rendered SSL snippet (`ssl.conf.j2`) enforcing TLS 1.2/1.3, strong ciphers, OCSP stapling, and HSTS is deployed to NGINX and validated with `nginx -t` before being applied.
4. **Reload** - Both certificate issuance and the SSL snippet notify the `reload nginx` handler, which only fires once per play run even if multiple tasks trigger it.
5. **Renew** - A cron job runs `certbot renew --quiet` daily, reloading NGINX only when a certificate was actually renewed (via Certbot's `--post-hook`).

## Design Decisions

- **Role-based structure** over a single flat playbook, so the certbot logic is reusable across other playbooks/projects.
- **`community.general`/`ansible.posix` collections** pinned in `requirements.yml` for reproducible CI runs.
- **Staging flag** (`certbot_staging`) lets you dry-run against Let's Encrypt's staging ACME endpoint to avoid production rate limits while testing.
- **`command` module for Certbot** was kept (rather than `community.crypto.acme_certificate`) to closely mirror how many teams actually automate this today; the trade-off is documented in the main README's "Lessons Learned" section.
