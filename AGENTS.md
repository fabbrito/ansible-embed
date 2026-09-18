# Agent guidelines

Ansible collection: roles and playbooks that converge Debian boards, plus the cloud-init seed that first-boots them.
Tested on Raspberry Pi OS Lite; keep roles Debian-generic and gate board-specific work on a fact. You never reach a
board; the dry-run and second converge are the consumer's, so write for them. `README.md` is the consumer contract,
`CONTEXT.md` the vocabulary, `docs/decisions.md` what is settled.

**Be concise, in English.** Code, comments, docs, commits: the fewest words that carry the fact.

## Workflow

1. Once per clone: `make deps && make hooks`. `make help` lists the rest.
2. Before changing a role, read its `defaults/main.yml` and the comments on the tasks you touch.
3. Touched a template: fixture for the branch, `make golden-update`, read the diff (`tests/README.md`).
4. Done: `make check` green and committed — the hooks enforce it and grade the message. `make test` and `make sanity`
   are release legs; `make release` runs them.

## Commits

- `type(scope): subject`, scope reused from `git log`. One change per commit.
- The hook grades the shape and the gate runs the lanes, both from a vendored engine reading `.githooks/hooks.conf`: a
  rule changes there, never in prose here.
- A file no lane in that config matches is never checked. A new kind of file means a new lane.
- AI co-authored: `Co-Authored-By:` naming the model. Never a session link.

> [!IMPORTANT] **No internal codes.** Plan-step, finding, severity (`P0`) or phase ids stay in scratch notes. Describe
> the thing, or define it in `docs/`. `make check-codes` sweeps.

## Releases

- `make release VERSION=x.y.z` stamps `galaxy.yml`, gates on every leg and tags; `make publish` sends it up. Both take
  `DRY_RUN=1`. There is no CI: a release is the only time the slow legs run.
- `galaxy.yml` `version:`, `CHANGELOG.md` heading and git tag move together — write the CHANGELOG section first, or
  `make release` refuses. Consumers pin tags, never branches.
- Contract change (new required var, unsafe default, renamed role): major, under "Changed".
- New authoring doc or repo-local tool path: add to `build_ignore` in the same commit. The build ignores `.gitignore`.
- Collection deps: `galaxy.yml` and `requirements.yml` together. Core floor: `meta/runtime.yml` and `scripts/lint.sh`
  together.

## Secrets

> [!IMPORTANT] **No secrets, board data or consumer data** — credentials, keys, hostnames, addresses, PSKs, locations —
> in code, docs or commits. Keep the fact, drop the name: _"a board behind a metered uplink"_, never the site.

- Task rendering a secret: `no_log: true`.
- Absent secret: skip. Assert only where absence is unsafe, and comment why.
- Keys documented in the role's `defaults/main.yml`, cross-cutting ones in the README table. Values never shipped.
- Defaults are empty, then asserted or skipped; say which.

## Roles

- One role per installable concern, kebab-case.
- Unprefixed vars are the consumer contract: read with `| default(...)`. Adding one: README table and `CHANGELOG.md`,
  same commit.
- Asserts check consumer input; goldens check our output. Neither replaces the other.
- Second converge reports zero changed: `creates:`, a `stat` guard, or `changed_when:` on every `command:`.
- `--check` survives: `when: not ansible_check_mode` where a prior install is fake; `check_mode: false` on read-only
  probes.
- No lock-out: sshd validated before reload; the tailnet is never the only door.
- Break-glass intact: cloud-init stays enabled, the seed stays on the card.
- Upgrades and their reboots run through unattended-upgrades, configured in `os`; no other role schedules either.
- Rendered files open with `# Rendered by Ansible — do not edit on host` and the source path.
- Playbook order sequences roles; no meta dependencies.

## Shell

- [YSAP style](https://style.ysap.sh), 80 columns.
- `set -uo pipefail` with explicit checks (`cd "$dir" || exit 1`); errexit never.
- `scripts/` and `.githooks/` need bash 4.4+, the hook engine's own floor; board-side shell may assume GNU userland.
- Make recipes delegate — to `scripts/`, or to the vendored engine. Logic never accumulates in Make syntax.

## Altitude

`code > comments > docs`. Altitude rises to the right; write at the lowest rung that holds the knowledge.

- **Code:** names, defaults, asserts, guards.
- **Comments:** focused on the line beside them; only what the code can't say — a hidden contract, a var's intent, a
  quirk's failure, the alternative that lost.
- **Docs:** knowledge spanning several sources in the repo. `docs/<topic>/` runbooks, `docs/decisions.md` what is
  settled, `docs/roadmap.md` where this is going; kebab-case. Point at roles and files, never paste them. The last two
  stay at repo altitude — a decision that only makes sense beside one file belongs in that file.

A comment reaching past its file belongs a rung up. A doc tied to one file belongs a rung down.

## Agent shell gotchas

- `ansible*` refuses non-blocking stdio and has no opt-out: the check runs at import, before any flag is read. This
  shell hands it a non-blocking stderr, so pipe the command through `cat` — `make check 2>&1 | cat`. Lanes cannot
  redirect, which is why the ansible legs sit in `scripts/lint.sh`.
- `cd` is wrapped by zoxide: use `builtin cd` or absolute paths.
