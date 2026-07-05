# Ansible Let's Encrypt Automation

![Ansible](https://img.shields.io/badge/ansible-%3E%3D2.14-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Production-style Ansible project that automates SSL/TLS certificate
provisioning and renewal for NGINX web servers using **Certbot** and
**Let's Encrypt** - including hardened SSL configuration, idempotent
certificate issuance, and automatic renewal via cron.

## Why this project

Manually renewing TLS certificates across a fleet of web servers doesn't
scale and is easy to forget. This project turns that into a single,
repeatable, idempotent Ansible run: install Certbot, issue certificates for
every configured domain, deploy a hardened NGINX SSL snippet, and set up
renewal so it never needs to be touched again.

## Architecture

See [docs/Architecture.md](docs/Architecture.md) for the full breakdown and
diagram. In short:

```
Ansible control node
        │
        ▼
 [webservers group] ──install──▶ certbot + plugins
        │
        ├──issue──▶ certbot certonly --nginx  (per domain, idempotent)
        │
        ├──harden──▶ deploy templates/ssl.conf.j2 (TLS 1.2/1.3, HSTS, OCSP stapling)
        │
        └──renew──▶ cron: certbot renew --quiet (daily @ 03:00)
```

## Repository Structure

```
ansible-letsencrypt-automation/
├── README.md
├── LICENSE
├── .gitignore
├── ansible.cfg
├── requirements.yml
│
├── inventory/
│   ├── hosts.ini
│   ├── group_vars/
│   │   └── webservers.yml
│   └── host_vars/
│
├── playbooks/
│   └── ssl_certbot.yml
│
├── roles/
│   └── certbot/
│       ├── defaults/main.yml
│       ├── vars/main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── install.yml
│       │   ├── certificates.yml
│       │   ├── nginx.yml
│       │   └── renewal.yml
│       ├── handlers/main.yml
│       ├── templates/ssl.conf.j2
│       ├── meta/main.yml
│       └── README.md
│
└── docs/
    ├── Architecture.md
    ├── Workflow.md
    └── Screenshots.md
```

## Features

- Modular `certbot` role (install / issue / harden / renew stages)
- Idempotent certificate issuance - safe to re-run, skips already-issued domains
- Supports multiple domains, each with its own SANs/aliases
- Staging-vs-production toggle (`certbot_staging`) to avoid Let's Encrypt rate limits while testing
- Hardened NGINX SSL configuration (TLS 1.2/1.3 only, strong ciphers, HSTS, OCSP stapling)
- Automatic renewal via cron with a reload-on-renew post-hook
- Fully tagged tasks (`certbot`, `ssl`, `nginx`, `install`, `renewal`)

## Quick Start

```bash
git clone https://github.com/<your-username>/ansible-letsencrypt-automation.git
cd ansible-letsencrypt-automation

ansible-galaxy collection install -r requirements.yml

# Edit inventory/hosts.ini and inventory/group_vars/webservers.yml
# with your real hosts and domains.

ansible-playbook -i inventory/hosts.ini playbooks/ssl_certbot.yml --check --diff
ansible-playbook -i inventory/hosts.ini playbooks/ssl_certbot.yml
```

See [docs/Workflow.md](docs/Workflow.md) for tagged runs and
[roles/certbot/README.md](roles/certbot/README.md) for the full variable reference.

## Lessons Learned / Design Trade-offs

This playbook deliberately uses the `command` module to shell out to
`certbot certonly --nginx`, guarded by `creates:` for idempotency. That
closely matches how many teams automate Let's Encrypt in practice, and
keeps the learning curve low.

In a stricter production environment, I'd consider swapping this for the
`community.crypto.acme_certificate` module (or a dedicated Certbot
Ansible Collection), which models the ACME protocol natively in Ansible
rather than shelling out to a CLI - giving better idempotency guarantees,
structured return data, and no dependency on the certbot binary being
installed at all. I kept the CLI-based approach here because it's what
most real-world teams run today and it keeps the project approachable.

## License

MIT - see [LICENSE](LICENSE).
