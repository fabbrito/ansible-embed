# Agent guidelines

Ansible collection: roles and playbooks that converge Debian boards, plus the cloud-init seed that first-boots them.
Tested on Raspberry Pi OS Lite; keep roles Debian-generic and gate board-specific work on a fact. You never reach a
board; the dry-run and second converge are the consumer's, so write for them. `README.md` is the consumer contract,
`CONTEXT.md` the vocabulary, `docs/adr/` the decisions.

**Be concise, in English.** Code, comments, docs, commits: the fewest words that carry the fact.

## Workflow

1. Once per clone: `make deps && make hooks`. `make help` lists the rest.
2. Before changing a role, read its `defaults/main.yml` and the ADRs covering it.
3. Touched a template: fixture for the branch, `make golden-update`, read the diff (`tests/README.md`).
4. Done: `make check` and `make test` green, change committed. Hooks enforce the first and grade the message.

## Commits

- `type(scope): subject`, scope reused from `git log`. One change per commit.
- AI co-authored: `Co-Authored-By:` naming the model. Never a session link.

> [!IMPORTANT] **No internal codes.** Plan-step, finding, severity (`P0`) or phase ids stay in scratch notes. Describe
> the thing, or define it in `docs/`. `make check-codes` sweeps.

## Releases

- `galaxy.yml` `version:`, `CHANGELOG.md` heading and git tag move together. Consumers pin tags, never branches.
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
- Reboots and upgrades belong to the operator; roles schedule neither.
- Rendered files open with `# Rendered by Ansible — do not edit on host` and the source path.
- Playbook order sequences roles; no meta dependencies.

## Shell

- [YSAP style](https://style.ysap.sh), 80 columns.
- `set -uo pipefail` with explicit checks (`cd "$dir" || exit 1`); errexit never.
- `scripts/` and `.githooks/` run on bash 3.2; board-side shell may assume GNU userland and bash 5.
- Make recipes delegate to `scripts/`.

## Altitude

`code > comments > docs`. Altitude rises to the right; write at the lowest rung that holds the knowledge.

- **Code:** names, defaults, asserts, guards.
- **Comments:** focused on the line beside them; only what the code can't say — a hidden contract, a var's intent, a
  quirk's failure, the alternative that lost.
- **Docs:** knowledge spanning several sources in the repo. `docs/<topic>/` runbooks, `docs/adr/` decisions,
  `docs/agents/` skill config; kebab-case. Point at roles and files, never paste them.

A comment reaching past its file belongs a rung up. A doc tied to one file belongs a rung down.

## Agent skills

- Issue tracker: GitHub issues via `gh` — `docs/agents/issue-tracker.md`.
- Triage labels: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix` —
  `docs/agents/triage-labels.md`.
- Domain docs: single context, root `CONTEXT.md` and `docs/adr/` — `docs/agents/domain.md`.
