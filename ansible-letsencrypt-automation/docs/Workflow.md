# Workflow

## Local Setup

```bash
ansible-galaxy collection install -r requirements.yml
```

## Running Against Real Hosts

```bash
# Dry run (staging ACME endpoint, no rate-limit risk)
ansible-playbook -i inventory/hosts.ini playbooks/ssl_certbot.yml \
  -e "certbot_staging=true" --check --diff

# Real run
ansible-playbook -i inventory/hosts.ini playbooks/ssl_certbot.yml
```

## Running a Subset with Tags

```bash
# Only install packages
ansible-playbook -i inventory/hosts.ini playbooks/ssl_certbot.yml --tags install

# Only redeploy the NGINX SSL snippet
ansible-playbook -i inventory/hosts.ini playbooks/ssl_certbot.yml --tags nginx
```
