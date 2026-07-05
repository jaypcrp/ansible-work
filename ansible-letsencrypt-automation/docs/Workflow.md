# Workflow

## Local Testing

```bash
# 1. Install dependencies
ansible-galaxy collection install -r requirements.yml
pip install ansible-lint yamllint molecule molecule-plugins[docker]

# 2. Lint
yamllint .
ansible-lint playbooks/ssl_certbot.yml

# 3. Run role tests in Docker via Molecule
cd roles/certbot
molecule test
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

## CI Pipeline

Every push/PR to `main` triggers [.github/workflows/ansible-lint.yml](../.github/workflows/ansible-lint.yml), which:

1. Installs Ansible, `ansible-lint`, and `yamllint`
2. Installs Galaxy collection requirements
3. Lints all YAML in the repo
4. Lints the playbook against the `production` ansible-lint profile
