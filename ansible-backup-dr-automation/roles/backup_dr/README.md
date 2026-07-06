# Ansible Role: backup_dr

Backs up PostgreSQL and MySQL databases plus application config directories,
uploads the archives to S3, and prunes local copies once they exceed the
retention window.

## Requirements

- `postgresql-client` on hosts in the `[databases]` group (for `pg_dump`)
- `mysql-client` on hosts in the `[mysql_servers]` group (for `mysqldump`)
- AWS CLI installed and on `PATH` on every backed-up host
- An IAM identity (access key/secret, or an instance role) with `s3:PutObject`
  on the target bucket

## Role Variables

See [defaults/main.yml](defaults/main.yml) for the full list. Key variables:

| Variable | Default | Description |
|---|---|---|
| `backup_root` | `/var/backups/ansible` | Local staging directory for backups |
| `s3_bucket` | `""` | Destination S3 bucket (must be set) |
| `retention_days` | `30` | Local backups older than this are deleted after upload |
| `postgres_databases` | `[]` | Databases dumped via `pg_dump` on `[databases]` hosts |
| `mysql_databases` | `[]` | Databases dumped via `mysqldump` on `[mysql_servers]` hosts |
| `config_dirs` | `[/etc/nginx, /etc/app]` | Directories archived on every host |
| `mysql_root_password` | `""` | MySQL root password - **source from vault**, never plaintext |
| `aws_access_key` / `aws_secret_key` | `""` | AWS credentials - **source from vault**, never plaintext |

## Host Group Behavior

Certain tasks are gated on inventory group membership (see [tasks/main.yml](tasks/main.yml)):

- `postgres.yml` only runs on hosts in `[databases]`
- `mysql.yml` only runs on hosts in `[mysql_servers]`
- `configs.yml` and `upload.yml` run on every host in the play

## Example Playbook

```yaml
- hosts: all
  become: true
  roles:
    - role: backup_dr
      s3_bucket: mycompany-backups
      retention_days: 30
```

## Tags

- `backup` - the entire role
- `prepare` - create the backup directory only
- `postgres` / `mysql` - database dumps only
- `configs` - config directory archive only
- `upload` - S3 sync only
- `cleanup` - retention pruning only

## Scheduling

This role does **not** install its own cron job on managed hosts - having a
backed-up server also own its own backup schedule creates a chicken-and-egg
risk (if the host goes down, so does the schedule). Instead, trigger
`playbooks/backup_dr.yml` on a schedule from wherever you already run
Ansible centrally, e.g.:

- A cron job on the Ansible control node / bastion
- An AWX/Ansible Automation Platform scheduled job template
- A CI/CD scheduled pipeline (GitHub Actions `schedule:`, GitLab scheduled pipelines, etc.)

See [docs/Workflow.md](../../docs/Workflow.md) for a concrete example.

## License

MIT
