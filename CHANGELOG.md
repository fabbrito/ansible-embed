# Changelog

All notable changes to `fabbrito.embed`. The version here, the version in `galaxy.yml`, and the git tag move together —
consumers pin the tag, so a change that is not released is a change nobody gets.

A released tag is never repointed.

## 1.0.0

Initial release. Each role adds its line in the commit that lands it.

- `baseline` playbook and `preflight` role: refuse an unsupported board before anything changes it.
- `os` role: weekly unattended upgrades with reboot when required, base packages, NTP servers, key-only sshd without
  root login.
- `seed` playbook and role: render a board's cloud-init `user-data` and `meta-data` from inventory.
