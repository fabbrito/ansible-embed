# Changelog

All notable changes to `fabbrito.embed`. The version here, the version in `galaxy.yml`, and the git tag move together —
consumers pin the tag, so a change that is not released is a change nobody gets.

A released tag is never repointed.

## 1.0.1

### Fixed

- `rclone`: removes `rclone.conf` when the R2 secrets are unset, instead of leaving the old credentials on the board.
- `os`: restarts timesyncd after an NTP change only if it was running.
- `preflight`: unreadable `cloud-init status` output fails with a message instead of a template error.
- `seed`: an inventory keyed by IP fails before the output directory is created.

### Changed

- Drops the unused `ansible.posix` dependency.
- `docs/rclone/upgrading.md` no longer ships in the collection.
- README: `target` documented, role sections regrouped, vocabulary fixed.

## 1.0.0

Initial release.

- `seed` playbook and role: render a board's cloud-init `user-data` and `meta-data` from inventory.
- `baseline` playbook and `preflight` role: refuse an unsupported board before anything changes it. Contract vars
  `embed_min_debian_major` and `embed_assert_hostname`.
- `os` role: weekly unattended upgrades with reboot when required, base packages, NTP servers, key-only sshd without
  root login.
- `rclone` role: pinned, checksum-verified upstream `.deb` per arch; `[r2]` remote and optional `[r2crypt]` wrapper from
  vault secrets.
- `docs/storage/`: runbook for attaching persistent USB storage.
