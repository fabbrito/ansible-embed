# Decisions

The choices this collection locked during design and shipped, and why the code has the shape it does. Repo altitude: the
reasoning behind the layer, not a restatement of it. The rules live in `CONTEXT.md`, `AGENTS.md` and `README.md`, and a
detail that only makes sense beside one role belongs in that role.

## What this layer owns, and how it travels

**Baseline only.** Service roles stay in the consuming repo, with the inventory, the secrets and the converge itself.
The alternative, a single collection holding the baseline and the services, loses: every consumer would carry services
it does not run, and a service's release cadence would ride on the baseline's.

**The tag is the artifact.** A consumer resolves a git tag over https: no branch, no registry, no vendoring, no
submodule. A tag installs without a credential, which is what lets an unattended runner adopt it, and it cannot move
under a consumer that has already pinned it. Registry publishing is deliberately absent; the release notes carry the
block a consumer pins.

**Builtins only, and a floor with no ceiling.** Every role uses `ansible.builtin`, so the collection has no dependency
edge and hands none to a consumer. ansible-core gets a minimum and no maximum: a ceiling lints clean, but it blocks
every consumer who upgrades core until this collection cuts a release, guarding against an incompatibility nobody has
seen.

## Which boards it accepts

**Debian, trixie or newer, and derivatives on its numbering.** The gate asserts the OS family and a major-version floor,
never the distribution. Ansible's parser maps Raspbian to Debian, so a distribution assert would refuse the reference
board.

**Arch means the userland.** Packages install for the userland, and on ARM it is not the kernel's architecture; the two
disagree on a 64-bit kernel with a 32-bit userland, and only the userland architecture is what these roles build for.

**ARMv6 is refused on the kernel architecture.** Raspbian's `armhf` userland runs on ARMv6, so the userland answer says
`armhf` on a board nothing here builds for; the kernel is the only fact that tells the truth there.

**Board-specific work is gated on a board fact, or left out.** A Pi-only collection was considered and rejected: the
baseline has nothing Pi-only in it, and a gate is a smaller thing to carry than a second product.

**A board's hostname must match its inventory name.** A card flashed for another board is otherwise silent: it
converges, just the wrong board. The check is therefore on by default, and an inventory keyed by address opts out.

## How a board first boots

**The seed is the bootstrap and the break-glass at once.** That is what rules out a root SSH key, and what rules out
ever disabling cloud-init: either would remove the only way back into a board nothing else reaches. The seed stays on
the card because it is the way back.

**A hand-made seed is accepted.** The converge asserts the seed's outcomes (cloud-init finished cleanly, the hostname,
key-only SSH, passwordless sudo) and never reads the file back. A card from the imager or a hand-written one is as valid
as ours if the board ends in the same state.

**The seed is deterministic.** Same inputs, same bytes; the identity a board carries is its name plus a generation, so
regenerating changes nothing and only a bump re-applies the seed. A timestamp identity, the imager's habit, would
re-seed a board on every accidental regeneration and churn every golden render.

## How a converge behaves

**An absent secret is a skip, not a failure.** A board converges before its secrets exist, which is what lets it join a
fleet first and receive credentials later. The cost is accepted: a misspelled key is indistinguishable from an absent
one, and a skipped task you expected to run is the only signal. A role asserts instead where absence leaves nothing to
do or leaves the board unsafe, and says why in place.

**The board maintains itself.** Upgrades and the reboots they sometimes require run only through unattended-upgrades,
configured by `os`; nothing else schedules either. A manual upgrade play was considered and rejected: a headless board
nobody logs into has to keep itself patched.

**Order lives in the playbook.** The baseline is an ordered list a consumer imports and appends its own plays to, and no
role declares a dependency on another. Meta dependencies would hide an order the baseline's correctness rests on.

**Storage is a manual step, not a role.** Attaching a USB stick is a one-time operator act (partition, format, mount by
UUID, guard the empty mount point), kept as a runbook. The collection neither formats nor mounts, because a role would
automate only the dangerous half for an operator who has to make the decisions anyway.

## What can be proven before a board

**The gate has three levels, and this repo can run one.** The lanes and release legs prove formatting, lint, the
rendered bytes and sanity. The two that tell the truth (a read-only dry-run against a real board with its diff read, and
a second converge reporting nothing changed) belong to the consuming repo, on the tag it adopts. This layer owns no
inventory and reaches no board, so it cannot run them.

**There is no CI.** The hooks gate each commit and the release command gates the tag; nothing runs on a push. The hook
engine is vendored and pinned, so an upstream release cannot turn this tree red and a bump is a one-file change.
