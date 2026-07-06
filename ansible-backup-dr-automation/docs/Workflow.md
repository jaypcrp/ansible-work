# Workflow

## Local Setup

```bash
# Encrypt your real secrets before ever committing them
cd inventory/group_vars/all
cp vault.yml.example vault.yml
$EDITOR vault.yml
ansible-vault encrypt vault.yml
```

## Running Against Real Hosts

```bash
ansible-playbook -i inventory/hosts.ini playbooks/backup_dr.yml --ask-vault-pass --check --diff
ansible-playbook -i inventory/hosts.ini playbooks/backup_dr.yml --ask-vault-pass
```

## Running a Subset with Tags

```bash
# Only dump databases, skip config archiving/upload/cleanup
ansible-playbook -i inventory/hosts.ini playbooks/backup_dr.yml \
  --ask-vault-pass --tags postgres,mysql

# Only prune old local backups
ansible-playbook -i inventory/hosts.ini playbooks/backup_dr.yml \
  --ask-vault-pass --tags cleanup
```

## Scheduling from the Control Node

Add a cron entry on the Ansible control node (not on the managed hosts)
so backups run automatically without the target servers depending on
their own health to trigger their own backups:

```cron
# /etc/cron.d/ansible-backup-dr
30 1 * * * ansible-ops cd /opt/ansible-backup-dr-automation && \
  ansible-playbook -i inventory/hosts.ini playbooks/backup_dr.yml \
  --vault-password-file /etc/ansible/.vault_pass >> /var/log/ansible-backup.log 2>&1
```

If you're using AWX / Ansible Automation Platform instead, wrap the same
command in a scheduled Job Template pointed at this playbook.
