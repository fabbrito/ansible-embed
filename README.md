# fabbrito.embed

Baseline convergence for Raspberry Pi OS Lite boards, as an Ansible collection.

It installs and keeps converged the layer every board needs and no board is interesting for: a platform and preseed
gate, OS hardening, conservative SD-card wear reduction, clock behaviour on a board with no real-time clock,
UUID-mounted USB storage, rclone against Cloudflare R2, and Tailscale for remote access. **Service roles do not live
here** — they stay in the repo that owns the service, which is also where inventory, secrets and the converge itself
live.

## Platform

|        |                                                                          |
| ------ | ------------------------------------------------------------------------ |
| OS     | Raspberry Pi OS **Lite**, bookworm (12) or newer                         |
| Init   | systemd                                                                  |
| Arch   | `armhf` or `arm64`, read from `dpkg --print-architecture` (the userland) |
| Boards | Raspberry Pi 2B and newer                                                |

ARMv6 boards — Pi 1, Zero, Zero W, CM1 — are **refused**: their userland needs `GOARM=5`, and nothing here builds for
it. Raspberry Pi OS is what this collection is tested and claimed against; other Debian-family ARM boards are
best-effort and unclaimed.

Not supported, and not planned: DietPi, Dropbear, Alpine/OpenWrt or any non-systemd init, and Debian before bookworm.

## Install

```yaml
# requirements.yml, in the consuming repo
collections:
  - name: git+https://github.com/fabbrito/ansible-embed.git
    type: git
    version: v1.0.0 # a tag, never a branch
```

The repo is public, so `https` needs no credential — which is what makes this installable from a CI runner without
handing it a deploy key. Declare where collections live **before** installing, or Ansible will not find what you just
installed and the FQCNs below fail to resolve:

```ini
# ansible.cfg, at the consuming repo's root
[defaults]
collections_path = ansible/collections
```

```bash
ansible-galaxy collection install -r requirements.yml
```

## Use

```yaml
# playbooks/site.yml, in the consuming repo
- import_playbook: fabbrito.embed.baseline

- name: Tailscale
  hosts: tailscale_hosts
  become: true
  roles:
    - { role: fabbrito.embed.tailscale, tags: [tailscale] }
```

This collection ships no inventory, no vault and no host — the contract it reads from the consumer is documented below
as each role lands.

## Development

```bash
make deps     # install the collections the roles depend on
make hooks    # enable the repo's git hooks (once per clone)
make check    # fmt-check + lint — the pre-commit gate
make test     # golden render tests
make sanity   # ansible-test sanity (slow on a cold venv)
```

## Security

This repo holds no secrets, no hostnames and no addresses, structurally. Report a vulnerability privately through the
Security tab — see [SECURITY.md](SECURITY.md).

## License

[Apache-2.0](LICENSE).
