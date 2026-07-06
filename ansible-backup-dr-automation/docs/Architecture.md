# Architecture

## Overview

This project automates backup and disaster-recovery (DR) for a mixed
PostgreSQL/MySQL/application-config environment, shipping encrypted archives
to S3 and pruning local copies once they age out of the retention window.

```
                     ┌─────────────────────────┐
                     │  Ansible control node   │
                     │  (or AWX / CI scheduler)│
                     └────────────┬─────────────┘
                                  │ SSH (triggered on a schedule)
                                  ▼
        ┌───────────────────────────────────────────────────┐
        │                    inventory: all                  │
        │  ┌───────────┐   ┌──────────────┐   ┌─────────────┐│
        │  │[databases]│   │[mysql_servers│   │[app_servers]││
        │  │ pg_dump    │   │ mysqldump    │   │ config only ││
        │  └───────────┘   └──────────────┘   └─────────────┘│
        └───────────────────────────────────────────────────┘
                                  │
                                  ▼
                     ┌─────────────────────────┐
                     │   Amazon S3 (SSE-AES256)│
                     └─────────────────────────┘
```

## Execution Flow

1. **Prepare** - ensure `/var/backups/ansible/<hostname>/` exists with `0700` permissions.
2. **Dump** - hosts in `[databases]` run `pg_dump` per database in `postgres_databases`; hosts in `[mysql_servers]` run `mysqldump` per database in `mysql_databases`. Both are gated by `group_names` so a host never attempts a dump engine it doesn't run.
3. **Archive configs** - every host tars/gzips its `config_dirs` (default `/etc/nginx`, `/etc/app`). Failures are captured and reported rather than silently ignored.
4. **Upload** - `aws s3 sync` ships the host's backup directory to `s3://<bucket>/<hostname>/<timestamp>/` with server-side encryption (SSE-AES256) and the `STANDARD_IA` storage class. Credentials are passed via `environment:`, never on the command line, and the task is `no_log: true` to keep secrets out of logs.
5. **Prune** - local archives older than `retention_days` are deleted with `ansible.builtin.find` + `file: state=absent`, so disks don't fill up once data is safely in S3. Remote-side retention is handled by an S3 lifecycle rule on the bucket, not by Ansible.

## Design Decisions

- **Idempotent dumps** - both `pg_dump` and `mysqldump` tasks use `creates:` guarded against the same timestamped filename, so re-running mid-failure doesn't re-dump data unnecessarily within the same run.
- **No cron installed on managed hosts** - scheduling lives on the control side (control-node cron, AWX, or CI), not on the servers being backed up. A host that's down doesn't take its own backup schedule down with it. See [roles/backup_dr/README.md](../roles/backup_dr/README.md#scheduling).
- **Secrets via vault, never inline** - `mysql_root_password`, `aws_access_key`, and `aws_secret_key` are sourced from an `ansible-vault`-encrypted file (`inventory/group_vars/all/vault.yml`), referenced indirectly via `vault_`-prefixed variables. Tasks touching them are `no_log: true`.
- **`command`/`shell` for dump tools** - there's no first-class Ansible module for `pg_dump`/`mysqldump`/`aws s3 sync`, so shelling out with `creates:`/`changed_when:` guards is the standard, idiomatic approach here (unlike the Certbot project, this isn't a trade-off to flag - it's the correct tool for the job).
