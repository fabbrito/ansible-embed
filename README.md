# fabbrito.embed

Baseline convergence for Debian boards, as an Ansible collection.

It installs and keeps converged the layer every board needs and no board is interesting for: a cloud-init seed for first
boot, a platform check, OS hardening with unattended upgrades, and rclone against Cloudflare R2. **Service roles do not
live here** — they stay in the repo that owns the service, which is also where inventory, secrets and the converge
itself live.

## Platform

|           |                                                                          |
| --------- | ------------------------------------------------------------------------ |
| OS        | Debian **13 (trixie)** or newer, or a derivative on Debian's numbering   |
| Init      | systemd                                                                  |
| SSH       | OpenSSH server                                                           |
| Arch      | `armhf` or `arm64`, read from `dpkg --print-architecture` (the userland) |
| Seed      | cloud-init, seeded from the boot partition                               |
| Reference | Raspberry Pi OS Lite trixie on a Raspberry Pi 2B — the one tested board  |

Other Debian boards are expected to work and are not tested. ARMv6 boards (Pi 1, Zero, Zero W, CM1) are **refused**:
Raspbian's userland runs there, but nothing here builds for `GOARM=5`.

Not supported: Ubuntu and other distros numbered apart from Debian, DietPi, non-systemd init, and Debian before trixie.

## Install

```yaml
# requirements.yml, in the consuming repo
collections:
  - name: git+https://github.com/fabbrito/ansible-embed.git
    type: git
    version: v1.0.1 # a tag, never a branch
```

Over `https` a public repo needs no credential — which is what makes this installable from a CI runner without handing
it a deploy key. Declare where collections live **before** installing, or Ansible will not find what you just installed
and the FQCNs below fail to resolve:

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

- name: My service
  hosts: my_service_hosts
  become: true
  roles:
    - { role: my_service, tags: [my_service] }
```

This collection ships no inventory, no vault and no host — the contract it reads from the consumer is documented below.

The baseline connects as `ansible_user` with passwordless sudo, which the seed grants.

### `preflight`

`preflight` runs first, under the `always` tag, and changes nothing: it refuses ARMv6, an unsupported OS or init, a
board without OpenSSH, a cloud-init that has not finished cleanly, and a hostname that does not match the inventory. A
disabled or absent cloud-init only warns: the seed can no longer recover that board.

#### Optional — absent, the default stands

| Var                      | Where            | What it buys                                                                                                         |
| ------------------------ | ---------------- | -------------------------------------------------------------------------------------------------------------------- |
| `embed_min_debian_major` | `group_vars/all` | Default `13`. Oldest Debian major version accepted.                                                                  |
| `embed_assert_hostname`  | `group_vars/all` | Default `true`. The hostname must equal the first label of the inventory name; `false` for an inventory keyed by IP. |

### `os`

Installs base packages and unattended-upgrades, sets NTP servers when given, and hardens sshd: keys only, no root login,
no X forwarding. Converge as the seeded user with a key; a password login is refused once sshd reloads.

Unattended upgrades run weekly from Debian, Raspbian and Raspberry Pi Foundation repos, and reboot at
`os_unattended_reboot_time` (board local time, the seed's timezone) only when an upgrade requires it. A dry-run on a
board without `python3-apt` fails at the package task; the first converge installs it.

#### Optional — absent, the default stands

| Var                         | Where            | What it buys                                                         |
| --------------------------- | ---------------- | -------------------------------------------------------------------- |
| `os_apt_packages`           | `group_vars`     | Default `[ca-certificates, avahi-daemon]`. Installed, never removed. |
| `os_unattended_origins`     | `group_vars`     | Default `[]`. Extra Origins-Pattern entries for other repos.         |
| `os_unattended_reboot_time` | `group_vars/all` | Default `"04:00"`, `HH:MM` local.                                    |
| `os_ntp_servers`            | `group_vars/all` | Default `[]`. systemd-timesyncd servers; empty leaves the image's.   |

### `rclone`

Installs a pinned upstream rclone (Debian's is too old for R2), verified by sha256, for `armhf` or `arm64`. Renders
`/root/.config/rclone/rclone.conf` when the R2 secrets are set, and removes it when they are not. Changing the rclone
version: `docs/rclone/upgrading.md`, in the repo.

#### Optional — absent, the role skips

| Var                                                                            | Where        | What it buys                                                                                              |
| ------------------------------------------------------------------------------ | ------------ | --------------------------------------------------------------------------------------------------------- |
| `rclone_r2_access_key_id`, `rclone_r2_secret_access_key`, `rclone_r2_endpoint` | vault        | The `r2` remote. All three or none, asserted.                                                             |
| `rclone_crypt_password`, `rclone_crypt_password2`                              | vault        | The `r2crypt` wrapper over `rclone_crypt_target`. Both or neither, asserted. `docs/rclone/encryption.md`. |
| `rclone_crypt_target`                                                          | `group_vars` | Default `r2:backups`. What `r2crypt` encrypts into.                                                       |

## Seed

A board first boots from a cloud-init seed on its boot partition. Render it from the same inventory the converge uses:

```bash
ansible-playbook fabbrito.embed.seed -e target=<board> -e seed_output_dir=<dir>
```

It writes `user-data` and `meta-data` into `<dir>`: a local directory to copy onto the boot partition (gitignore it), or
the mounted partition itself. The seed names the board after the first label of its inventory name and creates
`ansible_user` with passwordless sudo and your keys; no password, password SSH off, SSH enabled.

Break-glass: edit or regenerate the seed, bump `seed_generation`, boot. cloud-init applies it again, and the board's SSH
host keys change.

A hand-made seed works if it ends in the same state. Raspberry Pi Imager's "no passwordless sudo" writes `sudo: null`,
which strips the image's passwordless sudo and leaves nothing to converge with.

### Required by `seed`

| Var                    | Where                      | What it buys                                                                                   |
| ---------------------- | -------------------------- | ---------------------------------------------------------------------------------------------- |
| `target`               | `-e`                       | The one board to render. Unset matches no host.                                                |
| `ansible_user`         | inventory                  | The account the seed creates and converges connect as. Not root.                               |
| `seed_authorized_keys` | `group_vars` / `host_vars` | Public keys for that account. Asserted a non-empty list.                                       |
| `seed_output_dir`      | `-e`                       | Directory to write into, relative to the working directory; created when absent. Asserted set. |

### Optional — absent, the default stands

| Var               | Where            | What it buys                                               |
| ----------------- | ---------------- | ---------------------------------------------------------- |
| `seed_timezone`   | `group_vars/all` | IANA zone. Empty leaves the image's.                       |
| `seed_generation` | `host_vars`      | Default `1`. A new value re-applies the seed on next boot. |

## Storage

Application state belongs on a USB stick, not the SD card.
[`docs/storage/persistent-usb-storage.md`](docs/storage/persistent-usb-storage.md) attaches one by filesystem UUID: same
path in any port, and loud when the stick is absent.

## Development

```bash
make deps     # install the collections the roles depend on
make hooks    # enable the repo's git hooks (once per clone)
make check    # every hook lane over the working changes — the commit gate
make test     # golden render tests (a release leg)
make sanity   # ansible-test sanity (a release leg; slow on a cold venv)
make release  # stamp, gate, commit and tag — VERSION=x.y.z [DRY_RUN=1]
```

### Where the gate lives

There is no CI here. The hooks in your own clone gate every commit — formatting, a playbook syntax check and
`ansible-lint` at the production profile — and `make release` runs the slow legs once, before a tag exists: the golden
renders, `ansible-test sanity`, and an inspection of what the tarball would ship.

What this repo cannot prove is the half that matters most: a `--check --diff` against a real board with the diff read,
and a second converge reporting zero changed. This layer owns no inventory and reaches no host, so that gate is the
consuming repo's, on the tag it adopts.

## Security

This repo holds no secrets, no hostnames and no addresses, structurally. Report a vulnerability privately through the
Security tab — see [SECURITY.md](SECURITY.md).

## License

[Apache-2.0](LICENSE).
