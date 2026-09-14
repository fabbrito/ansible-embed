# Changelog

All notable changes to `fabbrito.embed`. The version here, the version in `galaxy.yml`, and the git tag move together —
consumers pin the tag, so a change that is not released is a change nobody gets.

A released tag is never repointed.

## 1.0.0

Initial release.

- `preflight` — the platform and preseed gate: userland arch from `dpkg`, board refusal below ARMv7, OS family and
  systemd and release floor, OpenSSH rather than Dropbear, cloud-init closed, non-interactive escalation, hostname
  matching inventory.
- `os` — the base layer: a minimal package set, `vm.swappiness`, an sshd hardening drop-in validated before it reloads,
  cloud-init neutralised, opt-in WiFi power-save, apt housekeeping. No `dist-upgrade`.
- `sdcard` — conservative wear reduction: a capped, persistent journal. `noatime` is opt-in and off.
- `time` — the consequences of having no RTC: time-sync asserted and waited on, and apt's date check disabled so a stale
  `fake-hwclock` cannot refuse a Release file.
- `storage` — USB state mounted by UUID, with a mount-is-real guard.
- `rclone` — arm-aware install from a pinned `.deb`, an R2 remote from vault credentials, an optional crypt wrapper.
- `tailscale` — group-scoped, additive to the LAN SSH path, joined only when an auth key exists.
