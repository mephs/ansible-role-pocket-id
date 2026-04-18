# ansible-role-pocket-id

[![Lint](https://github.com/mephs/ansible-role-pocket-id/actions/workflows/lint.yml/badge.svg)](https://github.com/mephs/ansible-role-pocket-id/actions/workflows/lint.yml) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE) [![Ansible Galaxy](https://img.shields.io/ansible/role/d/mephs/pocket_id?label=galaxy)](https://galaxy.ansible.com/ui/standalone/roles/mephs/pocket_id/) [![GitHub release](https://img.shields.io/github/v/release/mephs/ansible-role-pocket-id)](https://github.com/mephs/ansible-role-pocket-id/releases)

Ansible role that deploys [Pocket ID](https://pocket-id.org/) OIDC provider via Docker Compose.

## Requirements

- [community.docker](https://galaxy.ansible.com/ui/repo/published/community/docker) Ansible collection `>= 3.6.0` (see [`requirements.yml`](requirements.yml))

On the remote host:

- Docker Engine
- Docker Compose v2 plugin
- Python `requests` library

## Role Variables

You only need to set two variables:

| Variable | Description |
| --- | --- |
| `pocket_id_url` | Public URL at which Pocket ID will be reachable (`APP_URL`). |
| `pocket_id_encryption_key` | Encryption key for sensitive data at rest, min. 16 characters. See [Encryption key](#encryption-key) |

For the rest, see [`defaults/main.yml`](defaults/main.yml) for the full list and [`meta/argument_specs.yml`](meta/argument_specs.yml) for types, choices and descriptions.

## Encryption key

`pocket_id_encryption_key` must be at least 16 characters. Generate one with:

```bash
openssl rand -base64 32
```

### Encryption key rotation

To rotate the encryption key, run the following command inside the container:

```bash
docker compose exec pocket-id ./pocket-id encryption-key-rotate --new-key <new-encryption-key>
```

Then update `pocket_id_encryption_key` and re-run the role.

## Run behind reverse proxy

Pocket ID doesn't terminate TLS, so you'll want a reverse proxy (Traefik, Caddy, nginx) in front for any real deployment.

- Set `pocket_id_trust_proxy: true` so Pocket ID respects `X-Forwarded-*` headers from the proxy. Without it, logged client IPs will be wrong.
- If traffic passes through a CDN that rewrites the client IP (Cloudflare, Fly.io), set `pocket_id_trusted_platform` to the appropriate header (`CF-Connecting-IP`, `Fly-Client-IP`, etc.).

Two common topologies:

1. **Proxy on the same host.** Keep `pocket_id_network_external: false`. Port `pocket_id_port` is published on the host and the proxy reaches the app at `127.0.0.1:1411`.
2. **Proxy on a shared Docker network.** Set `pocket_id_network` to your proxy's network and `pocket_id_network_external: true`. No host port is published - the proxy reaches the container by name on the shared network. See the Traefik example below.

## Example playbooks

### Minimal

```yaml
- hosts: id
  become: true
  roles:
    - role: mephs.pocket_id
      vars:
        pocket_id_url: https://id.example.com
        pocket_id_encryption_key: "{{ vault_pocket_id_encryption_key }}"
        pocket_id_address: 0.0.0.0
```

### Behind Traefik on a shared external network

```yaml
- hosts: id
  become: true
  roles:
    - role: mephs.pocket_id
      vars:
        pocket_id_url: https://id.example.com
        pocket_id_encryption_key: "{{ vault_pocket_id_encryption_key }}"
        pocket_id_trust_proxy: true

        pocket_id_network: traefik
        pocket_id_network_external: true

        pocket_id_labels:
          traefik.enable: "true"
          traefik.http.routers.pocket-id.rule: "Host(`id.example.com`)"
          traefik.http.routers.pocket-id.entrypoints: "websecure"
          traefik.http.routers.pocket-id.tls.certresolver: "le"
          traefik.http.services.pocket-id.loadbalancer.server.port: "1411"
```

With `pocket_id_network_external: true` the `traefik` network must already exist and be attached to your Traefik container.

### With Cloudflare in front

```yaml
- hosts: id
  become: true
  roles:
    - role: mephs.pocket_id
      vars:
        pocket_id_url: https://id.example.com
        pocket_id_encryption_key: "{{ vault_pocket_id_encryption_key }}"
        pocket_id_trust_proxy: true
        pocket_id_trusted_platform: "CF-Connecting-IP"
        pocket_id_maxmind_license_key: "{{ vault_maxmind_license_key }}"
        pocket_id_address: 0.0.0.0
```

### With Prometheus metrics

```yaml
- hosts: id
  become: true
  roles:
    - role: mephs.pocket_id
      vars:
        pocket_id_url: https://id.example.com
        pocket_id_encryption_key: "{{ vault_pocket_id_encryption_key }}"
        pocket_id_metrics: true
        pocket_id_metrics_address: 127.0.0.1
        pocket_id_metrics_port: 9464
```

## License

[MIT](LICENSE)

## Author Information

Created and maintained by Mikhail Vorontsov (@mephs) <mvorontsov@tuta.io>
