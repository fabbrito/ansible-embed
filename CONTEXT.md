# Context

The vocabulary this repo uses, and what each word means _here_. It is a glossary and nothing else — the mechanics live
in the roles, the reasoning in `docs/adr/`, and the rules an author must uphold in `AGENTS.md`.

Use these words in issues, plans, comments, docs and role names. Where a near-synonym means something else here, the
entry says so.

## The two repos

**Layer**: This collection — the part of a board that every board needs and no board is interesting for. It owns roles
and playbooks, never an inventory, a vault, or a board. _Avoid_: framework, platform

**Consumer**: The repo that installs this layer and holds what it deliberately does not: inventory, secrets, service
roles, and the converge itself. There is more than one, and none of them may be named here. _Avoid_ as a name for this
repo: client, downstream, controller — "client" also means the authoring team's customer, and "downstream" the far side
of the edge, so all three stay legal in those senses.

**Contract**: The vars that cross roles and therefore carry no role prefix. The consumer sets them; this layer only
reads them. Adding one is a breaking change. _Avoid_: global var, shared var

**Pin**: The git tag a consumer resolves this collection at. Upgrades are a chosen act, so a pin never names a branch.
_Avoid_: version, dependency

## Boards

**Board**: A single-board computer running Debian or a derivative on Debian's numbering, plus the card it boots from and
any USB storage attached to it. The unit a consumer converges. The reference board is a Raspberry Pi on Raspberry Pi OS.
_Avoid_: device, node, machine

**Seed**: The cloud-init `user-data` and `meta-data` on a card's boot partition, which name the board and create its
first user. This layer renders it from the consumer's inventory; the consumer writes it to the card. The converge
asserts the seed's _outcomes_ and never reads the file back, so a hand-made seed is fine. It stays on the card, because
it is the break-glass. _Avoid_: preseed, firstrun, provisioning

**First boot**: The run of the seed that turns a flashed card into a reachable board. The seed is the bootstrap; there
is no bootstrap play here. _Avoid_: provisioning, setup

**Break-glass**: The way back into a board nothing else reaches: edit the seed on the card, bump its generation, boot.
cloud-init sees a new instance and applies the seed again. There is no root SSH key, and cloud-init is never disabled,
because either would take this away. _Avoid_: recovery, rescue

**Arch**: The **userland** architecture, as `dpkg --print-architecture` reports it — `armhf` or `arm64`. Not the
kernel's: a board running a 64-bit kernel with a 32-bit userland reports `arm64` there and `armhf` here, and only the
latter is what packages install for. _Avoid_: platform, machine, `uname -m`

**Operator**: The human running a converge and reading its output at 3am. Usually someone we have never met. _Avoid_:
user, admin

## Converging

**Converge**: One run of a play against a board, driving it to its declared end state. _Avoid_: deploy, provision
(deploying a service is the consumer's word for its own roles)

**Converged end state**: A board that is correct and finished — including one that skipped work whose secret was absent.
Skipped is not degraded. _Avoid_: partial, degraded

**Baseline**: The ordered set of roles every board gets. The order is load-bearing, not stylistic.

**Group-scoped role**: A role deliberately outside the baseline, which a board runs iff its inventory puts it in that
group. One line of inventory instead of a conditional inside a role. _Avoid_: optional role, conditional role

**Load-bearing**: Of an order, a field name, a default: change it and something breaks silently, elsewhere. Marks the
places where "it reads better this way" is a bug report. _Avoid_: important, critical

**Lock-out**: Losing the SSH path to a board — a converge severing Ansible's own connection, which does not fail loudly
(the task succeeds and the board is gone), or a hardening change that leaves no way back in. Guarded ahead of time;
break-glass is the recovery, not the plan. _Avoid_: outage, bricking

## Storage and wear

**Wear**: Write amplification on the SD card, which is what ends a cheap card's life. The baseline reduces it
conservatively: the journal is capped and kept persistent. The aggressive tier — tmpfs logs, a volatile journal, a
read-only root — is a decision we have not taken. _Avoid_: optimization, tuning

**Storage volume**: A USB stick or disk the consumer declares, mounted at a path by UUID. The unit `storage` takes as
input; formatting and partitioning are a manual step, never this layer's. _Avoid_: disk, drive, mount

**Mount-is-real guard**: The check that a path is genuinely a mount point after mounting it. Without it a missing stick
leaves writes landing silently on the SD card — the failure is invisible until the card fills. _Avoid_: verification,
sanity check

## Remote access

**Tailnet**: The private network Tailscale joins a board to. It is additive: the LAN SSH path stays reachable and the
tailnet is never the only door to a board. _Avoid_: VPN, overlay

**Auth key**: The credential a board joins its tailnet with. Absent means installed but not joined — a skip, not a
failure. _Avoid_: token, secret key

## The gate

**Gate**: The three checks that stand between a change and a converged board, in increasing order of truthfulness. Only
the first runs here. _Avoid_: test suite, CI (there is no suite; naming one oversells it)

**Dry-run**: A check-mode converge against a real board, read for its diff. Not proof, because check mode lies where a
prerequisite was never really installed — but an unexpected diff is always real. _Avoid_: simulation, preview

**Second converge**: Running the play twice and requiring the second run to report zero changed. The closest thing to a
real test that exists here.

**Changed-every-run**: A task that reports changed on every converge. A bug even when the board ends up correct, because
real drift then hides in the noise. _Avoid_: noisy, non-idempotent

**Precondition**: An invariant a role depends on that the variable schema cannot make unrepresentable, and that fails
silently if unguarded. Roles assert these and nothing else — never that a package installed. _Avoid_: check, outcome
(ADR-0010 uses "validate" for the act; the noun for the thing asserted is precondition)

## Secrets and absence

**Absent-secret skip**: The default behaviour: a role missing its secret does less, quietly, and still converges. It is
what lets a board join a fleet before its secrets exist. _Avoid_: graceful degradation, fallback

**Assert instead of skip**: The exception, taken where absence leaves nothing to do or leaves the board unsafe. A role
choosing it says why, in place.

**Silent skip**: The cost of the rule — a misspelled secret key is indistinguishable from an absent one, and a skipped
task you expected to run is the only signal.

## Backups

**Remote**: A named rclone destination. The baseline installs rclone on every board and renders the remote only where
the R2 credentials exist. Renaming it is a breaking change consumers' backup roles feel, because they address it by
name.

**Crypt wrapper**: The optional encryption layer over the backups remote, which services opt into by pointing at it
instead. Lose its passwords and the backups are unrecoverable — no escrow, no support ticket. _Avoid_: encrypted remote
