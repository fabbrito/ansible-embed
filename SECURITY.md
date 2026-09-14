# Security policy

## Reporting a vulnerability

Report privately, not as a public issue: use GitHub's **Report a vulnerability** button under this repo's Security tab.
That channel is private until a fix is published, and it is the only one — there is no mailbox to fall back to. Expect
an acknowledgement within a week.

Please do not include anything from the board you found it on — a hostname, an address, a key, a log line with a token
in it. Describe the role, the variables involved, and what converged wrong. This repo holds no secrets and no board data
by design, and a report is not an exception to that.

## What is in scope

This collection converges boards. A finding is in scope when a documented use of a role leaves a board less safe than
the role claims. The recurring shapes:

- **A default that is unsafe to inherit.** Roles default to empty and then skip or assert; a default that carries a real
  value someone else's board would silently adopt is a bug even if it is safe for one board.
- **A rendered secret reaching the play output.** Every task that renders one carries `no_log: true`. A path that
  bypasses it — a diff, a loop label, a failed assert's message — is a vulnerability, because it lands in the consumer's
  CI logs.
- **A converge that can lock the operator out**, or that opens a door before the thing guarding it exists.
- **A guard that passes when the thing it guards is absent** — a mount recorded as mounted while the writes land on the
  SD card, an assert satisfied by a stale fact.
- **A hardening drop-in that renders valid and says the wrong thing** — `sshd` accepting a config whose effective value
  is not what the role claims.

Out of scope: vulnerabilities in the upstream packages these roles install (report those upstream), and the security of
a consuming repo's inventory, vault, or CI — none of which lives here.

## Supported versions

The most recent tag only. Consumers pin a tag, so a fix ships as a new tag they adopt deliberately; there are no
backports to older ones and no branch to track. A released tag is never repointed — if a fix has to reach you, it
reaches you as a version you can see in your own `requirements.yml` diff.
