# Ansible Role: certbot

Provisions and auto-renews Let's Encrypt SSL certificates for NGINX using Certbot.

## Requirements

- Ubuntu/Debian target hosts (uses `apt`)
- NGINX already installed and serving the domains listed in `certbot_domains`
- Port 80/443 reachable from the internet for HTTP-01 validation

## Role Variables

See [defaults/main.yml](defaults/main.yml) for the full list. Key variables:

| Variable | Default | Description |
|---|---|---|
| `certbot_email` | `admin@example.com` | Contact email registered with Let's Encrypt |
| `certbot_domains` | `[]` | List of `{name, aliases[]}` domain objects to issue certs for |
| `certbot_staging` | `false` | Use the Let's Encrypt staging environment (avoids rate limits while testing) |
| `nginx_ssl_snippet_path` | `/etc/nginx/snippets/ssl.conf` | Where the hardened SSL snippet is deployed |
| `certbot_renewal_hour` / `certbot_renewal_minute` | `3` / `0` | Cron schedule for automatic renewal |

## Example Playbook

```yaml
- hosts: webservers
  become: true
  roles:
    - role: certbot
      certbot_email: "admin@example.com"
      certbot_domains:
        - name: example.com
          aliases: [www.example.com]
```

## Tags

- `certbot` - install packages, issue certificates, configure renewal
- `ssl` - certificate issuance and SSL config only
- `nginx` - SSL snippet deployment only
- `install` - package installation only
- `renewal` - cron renewal job only

## License

MIT
