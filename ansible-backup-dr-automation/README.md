# Ansible Automated Backup & Disaster Recovery

![Ansible](https://img.shields.io/badge/ansible-%3E%3D2.14-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Production-style Ansible project that automates database and configuration
backups across a mixed PostgreSQL/MySQL environment, ships encrypted
archives to S3, and enforces a retention policy - so backups happen on a
schedule instead of relying on someone remembering to run a script.

## Why this project

Backups that depend on a human remembering to run them are backups that
eventually don't happen. This project turns that into a single, idempotent
Ansible run: dump the right databases on the right hosts, archive
application configs, upload everything to S3 with server-side encryption,
and prune local copies once they're safely offsite and past their
retention window.

## Architecture

See [docs/Architecture.md](docs/Architecture.md) for the full breakdown and
diagram. In short:

```
Ansible control node (triggered on a schedule)
        │
        ▼
 [databases] ──pg_dump──▶ gzip ──▶ backup_root/<host>/
 [mysql_servers] ──mysqldump──▶ gzip ──▶ backup_root/<host>/
 [all hosts] ──archive──▶ config_dirs ──▶ backup_root/<host>/
        │
        ├──upload──▶ aws s3 sync  (SSE-AES256, STANDARD_IA)
        │
        └──cleanup──▶ delete local backups older than retention_days
```

## Repository Structure

```
ansible-backup-dr-automation/
├── README.md
├── LICENSE
├── .gitignore
├── ansible.cfg
│
├── inventory/
│   ├── hosts.ini
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── all/
│   │       └── vault.yml.example
│   └── host_vars/
│
├── playbooks/
│   └── backup_dr.yml
│
├── roles/
│   └── backup_dr/
│       ├── defaults/main.yml
│       ├── vars/main.yml
│       ├── tasks/
│       │   ├── main.yml
│       │   ├── prepare.yml
│       │   ├── postgres.yml
│       │   ├── mysql.yml
│       │   ├── configs.yml
│       │   ├── upload.yml
│       │   └── cleanup.yml
│       ├── handlers/main.yml
│       ├── meta/main.yml
│       └── README.md
│
└── docs/
    ├── Architecture.md
    └── Workflow.md
```

## Features

- Modular `backup_dr` role (prepare / dump / archive / upload / cleanup stages)
- Group-aware: `pg_dump` only runs on `[databases]`, `mysqldump` only on `[mysql_servers]`
- Idempotent dumps guarded by `creates:` on a timestamped filename
- Config directories archived on every host via `ansible.builtin.archive`
- S3 upload with SSE-AES256 encryption and configurable storage class
- Secrets (DB password, AWS keys) sourced from an `ansible-vault`-encrypted file, never plaintext
- Retention-based local cleanup via `ansible.builtin.find` + `state=absent`
- Fully tagged tasks (`backup`, `prepare`, `postgres`, `mysql`, `configs`, `upload`, `cleanup`)

## Quick Start

```bash
git clone https://github.com/<your-username>/ansible-backup-dr-automation.git
cd ansible-backup-dr-automation

# Set up your vaulted secrets
cd inventory/group_vars/all
cp vault.yml.example vault.yml
$EDITOR vault.yml
ansible-vault encrypt vault.yml
cd -

# Edit inventory/hosts.ini and inventory/group_vars/all.yml
# with your real hosts, databases, and S3 bucket.

ansible-playbook -i inventory/hosts.ini playbooks/backup_dr.yml --ask-vault-pass --check --diff
ansible-playbook -i inventory/hosts.ini playbooks/backup_dr.yml --ask-vault-pass
```

See [docs/Workflow.md](docs/Workflow.md) for tagged runs and control-node
scheduling, and [roles/backup_dr/README.md](roles/backup_dr/README.md) for
the full variable reference.

## Lessons Learned / Design Trade-offs

The role deliberately does **not** install a cron job on the servers being
backed up. Putting the schedule on the same host that's being backed up
creates a circular dependency - if that host goes down hard enough to need
a restore, its own backup schedule went down with it. Instead, scheduling
lives on the control side (a cron job on the Ansible control node, an
AWX/AAP scheduled Job Template, or a CI pipeline's `schedule:` trigger),
which is documented in [roles/backup_dr/README.md](roles/backup_dr/README.md#scheduling).

Secrets (`mysql_root_password`, AWS keys) are deliberately kept out of
`group_vars/all.yml` and sourced instead from a separate `ansible-vault`-encrypted
file, with all tasks that touch them marked `no_log: true` - so a
`--check --diff` or verbose run never prints them.

## License

MIT - see [LICENSE](LICENSE).
