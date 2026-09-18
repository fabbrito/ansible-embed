# Roadmap

Where this collection intends to grow, at repo altitude: proposals, and what is already known about them. Nothing here
is scheduled or decided: dates, task lists, and the investigation a proposal needs all arrive when the work is planned.
`decisions.md` records what shipped, and a proposal that lands updates it then.

## Next

**`tailscale`.** A group-scoped role: a board gets it iff its inventory puts it in that group, and it installs without
joining when its auth key is absent. The tailnet is additive: the LAN path stays reachable, and the tailnet is never the
only door.

## Proposals

**A role for USB storage.** Pin an attached stick at a fixed path by filesystem UUID, and make the empty mount point
immutable so writes fail while the stick is absent instead of landing on the SD card. Formatting stays the operator's
act; the runbook's manual steps are what the role would take over.

**WiFi in the seed.** Only if a board arrives that needs it. It pays a known cost: the PSK would sit on a FAT partition
any local user can read, which collides with the seed staying on the card as break-glass. The imager's soft-block lines
return with it.

**armv6 support.** ARMv6 boards are refused today because nothing here builds for `GOARM=5`. Supporting them means an
armv6 rclone asset and lifting that refusal.

**Hardware watchdog.** Enable `RuntimeWatchdogSec` on a board that exposes `/dev/watchdog`, gated on the device
existing.
