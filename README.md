# Ansible Role: certs

[![CI](https://github.com/tjg-homelab/ansible-role-certs/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-certs/actions/workflows/ci.yml)

Distributes a TLS certificate and key from a single source to many Linux hosts.
Think of it as a **certbot fan-out manager**: one host renews (or already holds)
the cert, and this role copies it everywhere it's needed — including delegated
hosts and fixed-path symlinks — in one pass.

It does **not** issue certificates; it moves an existing cert/key around. Pair it
with certbot (or any issuer) on the source host.

## How it works

Pick a source mode:

| `certs_source_mode` | Source | Use case |
|---|---|---|
| `certbot_host` | One managed host's `/etc/letsencrypt/live/...` (optionally `certbot renew` first) | The normal renew-and-distribute flow |
| `controller_fixture` | Files on the Ansible control node | Air-gapped/offline distribution and tests |
| `nfs_share` | A path already mounted on each target | Certs staged on shared storage |

The role loads the cert/key once from the source, then on every play host:
installs them into `certs_path` / `certs_key_path`, optionally copies them to extra
hosts (`certs_delegate_copies`), and optionally creates symlinks (`certs_symlinks`).

## Requirements

- Debian 12/13, Ubuntu 22.04/24.04, or Enterprise Linux 9
- For `certbot_host` mode: a working certbot on the source host
- Uses only `ansible.builtin` modules

## Key Role Variables

| Variable | Default | Description |
|---|---|---|
| `certs_source_mode` | `certbot_host` | `certbot_host` \| `controller_fixture` \| `nfs_share` |
| `certs_source_host` | `""` (first play host) | Host holding the source cert |
| `certs_domain_name` | `example.com` | Drives certbot live-dir + default filenames |
| `certs_renew_before_distribute` | `true` | Run `certs_renew_command` on the source first |
| `certs_manage_dns_credentials` | `false` | Write a certbot DNS-plugin creds file for the renew, then delete it |
| `certs_dns_credentials_content` | `""` | Plugin-agnostic creds file body (vault it) |
| `certs_path` / `certs_key_path` | `/etc/ssl/certs` / `/etc/ssl/keys` | Destination dirs |
| `certs_target_cert_filename` | `{{ certs_domain_name }}.pem` | Installed cert name |
| `certs_target_key_filename` | `{{ certs_domain_name }}.key` | Installed key name |
| `certs_delegate_copies` | `[]` | Extra per-host copies (see `defaults/`) |
| `certs_symlinks` | `[]` | Symlinks pointing at the deployed cert/key |

See `defaults/main.yml` for the full set and inline examples.

### DNS-plugin credentials (any certbot plugin)

`certs_dns_credentials_content` is written verbatim, so it works with any certbot
DNS plugin. Cloudflare example:

```yaml
certs_manage_dns_credentials: true
certs_dns_credentials_path: /root/.secrets/certbot/cloudflare.ini
certs_dns_credentials_content: "dns_cloudflare_api_token = {{ vault_cloudflare_token }}"
```

## Example Playbook

```yaml
- hosts: web
  roles:
    - role: certs
      vars:
        certs_source_mode: certbot_host
        certs_source_host: cert-master.example.com
        certs_domain_name: example.com
        certs_symlinks:
          - src: "/etc/ssl/certs/example.com.pem"
            dest: /etc/gitlab/ssl/example.com.crt
          - src: "/etc/ssl/keys/example.com.key"
            dest: /etc/gitlab/ssl/example.com.key
```

Installing via `requirements.yml`:

```yaml
roles:
  - name: certs
    src: https://github.com/tjg-homelab/ansible-role-certs.git
    version: v1.0.0
```

## Testing

Molecule (Docker driver) generates a throwaway self-signed cert/key on the control
node at runtime (nothing cert-shaped is committed), distributes it via
`controller_fixture` mode to two containers, and verifies the installed files and
modes, the symlinks, and a delegated fan-out copy to a second host.

```bash
pip install ansible-core molecule molecule-plugins[docker] docker
ansible-galaxy collection install community.docker ansible.posix
molecule test
```

## License

MIT

## Author

Rodney Nissen ([The Jira Guy](https://thejiraguy.com)) — Senior Atlassian
Consultant & Jira Architect.
