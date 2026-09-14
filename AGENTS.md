# Conventions

This is an Ansible **collection**. It ships the roles and playbooks that converge Raspberry Pi OS Lite boards; it does
not own an inventory, a vault, or a board, and never reaches one. Everything below follows from that: the artifacts are
roles, the gate is a static `make check`, and what can go wrong is somebody else's board on somebody else's converge.
The consuming repo holds inventory, secrets, service roles, and the converge — `README.md` has the seam between the two,
`CONTEXT.md` the vocabulary.

## Platform

**Raspberry Pi OS Lite, bookworm (12) or newer**, on Raspberry Pi 2B and newer, systemd, both bitnesses. GNU userland
and bash 5 are a given: `${var,,}`, `sort -V`, `grep -P` and friends are fair game. Raspberry Pi OS is what we test and
claim; other Debian-family ARM SBCs are best-effort and unclaimed.

The userland architecture is read from **`dpkg --print-architecture`**, never `uname -m` or `ansible_architecture`:
Ansible has no userland-arch fact on ARM, and a board running a 64-bit kernel with a 32-bit userland misreports there.
`ansible_distribution` also differs by bitness — 32-bit Raspberry Pi OS is Raspbian (`ID=raspbian`), 64-bit is Debian —
so gates assert on `ansible_os_family`, never the distribution id.

## Language

All developer-facing text is **English** — comments, commit messages, variable names, docs.

## Naming

- **Roles** are kebab-case, named for the thing they install or the concern they own (`sdcard`, `time`, `tailscale`).
  One role per installable concern.
- **Role variables are prefixed with their role** — `storage_volumes`, `os_wifi_powersave`, `time_ntp_servers`.
  Ansible's var namespace is flat and has no scoping; the prefix is what keeps two roles from colliding.
  - `ansible-lint` enforces it (`var-naming[no-role-prefix]`), `register:` and `set_fact:` included.
- **Vars that cross roles carry no prefix and are a CONTRACT, not a default.** A collection cannot ship `group_vars`, so
  `embed_min_debian_major` and `embed_arch_override` are set by the consumer and merely consumed here. Adding one is a
  breaking change to that contract: it goes in the README table and in `CHANGELOG.md`, in the same commit.
- **Files** follow Ansible's layout, which is load-bearing: `tasks/main.yml`, `defaults/main.yml`, `handlers/main.yml`,
  `templates/*.j2`. Ansible reads these paths; they are not free-form.
- Markdown stays kebab-case.

## Nothing in here names a consumer

This layer is shared across boards and operators. A comment, a default, or a doc naming a client, a client's vendor, a
domain, a board other than the reference one, what a board is used for, or a consuming repo has leaked — wrong here even
when accurate.

**Keep the fact, drop the name:** _"a board behind a metered uplink"_ carries the same warning as the site's name and
travels. Same for defaults — one encoding a board's hostname or a network's credentials is a bug the next consumer
inherits silently. Default to empty and assert, or default to empty and skip; say which. A WiFi PSK, a public address,
or a key never belongs here, in any form, including in a doc.

## Altitude

Where knowledge lives, in order: `code > comments > docs`. Moving right raises altitude; write at the lowest level that
holds the knowledge. Local to one file is a comment; spanning roles or files is a doc. A comment explaining another role
is a doc in the wrong place.

## Comments

A comment carries what the YAML can't, in as few words as it takes. Sequencing is most of this repo's logic and almost
none of it is visible in the tasks, so these four are worth writing down — tersely.

- **Hidden contracts** — what a task assumes, what the next one needs. `preflight` proves the seed applied before `os`
  disables cloud-init; `storage` mounts before anything writes to the mount point. Reorder either and the board breaks.
- **Interface intent** — what a role or a var is _for_. `defaults/main.yml` is the role's public API, and here it is the
  API a team you will never talk to reads: what setting a var buys, not that it has a default.
- **Quirks** — the guard, the workaround, the landmine. `when: not ansible_check_mode`, `creates:`, `sshd -t` before the
  handler reloads, `Acquire::Check-Date "false"` under a stale `fake-hwclock`: each is a failure we hit. Name the
  failure, not the story.
- **Why, not how** — the alternative and why it lost: reading the arch from `dpkg` over `uname -m`, a persistent journal
  over a volatile one, the pinned rclone `.deb` over a zip into `/usr/local/bin`.

Do not restate the module, label the obvious, repeat the task's own `name:`, narrate, or recount a comment's own
history. A line or two; if it genuinely needs more it is a `docs/` page or an ADR, and the comment points at it.

## Docs

`docs/` splits three ways: `docs/<topic>/` holds **runbooks**, `docs/adr/` holds **decisions**, `docs/agents/` holds
**skill configuration**. The rules below are the runbook rules.

A runbook is a procedure a human follows, in order, to get a result — read by operators of boards we do not run, so it
describes the role's mechanics and never a particular board.

- **Stay operational.** A runbook is steps and their rationale, not a transcription of what the roles do. The roles are
  the source of truth for mechanics.
- **Point, don't pin.** Reference roles and files as navigation pointers. Never cite line numbers, and never paste task
  YAML into a doc — it goes stale the moment the role changes.
- **Why over how.** Capture the reasoning that made an approach right, so a reader can tell when it stops being right.

Ask: will this still be true after a refactor that preserves behavior? If not, it belongs in a code comment next to the
thing, not in a doc.

## Commits

Commits follow `type(scope): subject`.

- **Scope is the role or layer** — `os`, `sdcard`, `time`, `ci`, `galaxy`. Reuse a scope the history already reaches for
  (`git log --format='%s'`) before coining one; omit it when the change is genuinely repo-wide.
- **Subject** — concise and imperative; name what the commit did, not how big it was.
- **Bias hard to terse.** Subject-only by default; a body is short bullet topics, never prose. `commit-msg` grades the
  shape and prints it on rejection — don't work around a cap, the commit that doesn't fit should have been two.
- **One commit per change.** Each fix or refactor is atomic and independently revertable.
- **Green between commits.** Every commit leaves `make check` passing; `pre-commit` enforces it.
- If an AI co-authored, end with a `Co-Authored-By:` trailer naming the model, after a blank line. Never a session URL
  or any other link into a private session.

> [!IMPORTANT] **No internal codes.** Ids coined while working — review-finding ids, plan-step ids, severity labels
> (`P0`), phase labels — never reach a commit message, doc, comment, or issue: a reader without your scratch notes
> cannot resolve them. **Strip** the label (describe the thing) or **promote** it (define it in `docs/`, after which it
> resolves). Real-world ids (CVE, RFC) are fine. `make check-codes` sweeps for them.

> [!IMPORTANT] **No secrets, no board data, no consumer data.** This repo has no vault and must never acquire one. A
> credential, a certificate, a private key, a hostname, a WiFi passphrase, or a public address belonging to any board
> does not belong here — not in code, not in a commit message, not in a doc. `.gitignore` keeps the usual files out;
> review catches the rest.

## Releases

The consumer pins a git tag, so **a change that is not released is a change nobody gets.**

- `galaxy.yml`'s `version:`, the `CHANGELOG.md` heading, and the git tag move together, in one commit plus one tag.
- A change to the contract in `README.md` — a new required var, a default that stopped being safe, a renamed role — is a
  **major** bump and says so in the changelog under "Changed".
- Never point a consumer at a branch. The pin is the whole safety mechanism.
- **A new authoring doc or repo-local tooling path joins `build_ignore` in the same commit.** It is the only exclusion
  list the build reads — `.gitignore` is not consulted — so anything unnamed ships into a consumer's tree, where their
  own agents read it. `.tmp/` is the standing example: gitignored, and named in `build_ignore` anyway.

## Secrets

This repo holds none, structurally: no `vault.yml`, no `.vault_pass`, no inventory to attach them to. What lives here is
how roles _behave_ around a consumer's secrets.

- **Any task that renders a secret carries `no_log: true`.** Without it the value lands in the play output and in the
  consumer's CI logs.
- **A role whose secret is absent skips its work, it does not fail.** Preserve it in new roles; the reasoning and the
  worked examples are ADR-0001's.
- **Except where absence is itself unsafe** — or where doing less quietly would change what converged rather than shrink
  it. When you assert rather than skip, say why in the comment; the default is to skip.
- A role documents its expected keys in its own `defaults/main.yml`, and the cross-cutting ones in the README table.
  Documenting a key is not the same as shipping it; never ship a value.

## Idempotence and safety

The invariants that make a converge re-runnable. **No tool here checks any of them** — they hold because you follow
them.

- **Every role is safe to re-run.** A second converge on an already-converged board changes nothing. A task that can't
  express this natively gets `creates:`, a `stat` guard, or an explicit `changed_when:` — never a blind `command:` that
  reports changed every run.
- **A converge must not be able to lock the operator out.** Validate the sshd config before the handler reloads; never
  let the tailnet become the only door to the board.
- **`--check` must survive.** A task that cannot run in check mode (a prior task's package isn't really installed) is
  gated `when: not ansible_check_mode`, so the consumer's dry-run reports cleanly instead of erroring. A broken guard
  surfaces only in the consuming repo, which is why it is a rule.
- **No scheduled OS upgrades, and no reboot the converge performs on a schedule.** An appliance carries no
  `dist-upgrade` timer; a reboot is an operator's act, not a role's.
- **Rendered files announce themselves** — `# Rendered by Ansible — do not edit on host`, plus the source path. Someone
  finds the file at 3am and needs to know editing it is pointless.
- **Service roles declare no meta dependencies.** Baseline roles run first, by playbook order; meta deps would re-walk
  `os` on every service deploy.

## Verification

There is no unit-test suite. The gate here is:

```bash
make check    # fmt-check + lint — must be green to commit
make sanity   # ansible-test sanity — CI runs it; slow on a cold venv
make test     # golden render tests — CI runs it
```

- **`make check` is static** — formatting, playbook syntax, `ansible-lint` (must stay clean at the **production**
  profile), `shellcheck`, and a collection build. It touches no board. FQCN resolution is part of it, so a role renamed
  without updating `playbooks/baseline.yml` fails there.
- **`make sanity` is out of `check` on cost, not on importance** — a cold run builds a venv per supported Python. CI
  runs it. Its staging is delicate and `scripts/sanity.sh` says why, including the guard that fails a silent all-skip.
- **`make test` pins the rendered BYTES of the templates.** Asserts validate the consumer's input; goldens validate our
  output, and the gap between them is where this repo's worst bugs live — a config that is valid and says the wrong
  thing passes both `ansible-lint` and the tool's own validator. **A template change with no golden diff means you
  changed nothing or you have no fixture for the branch you touched**; adding a branch means adding a fixture, in the
  same commit. Layout is in `tests/README.md`.
- **The gate that matters is the consumer's and you cannot run it.** A `--check --diff` dry-run against a real board,
  and a second converge reporting zero changed, both belong to the consuming repo.

# Tooling

- **Make is the entrypoint**, and it is thin on purpose: it delegates to `scripts/` and `ansible-galaxy`. Real logic
  lives in roles and scripts, never in a recipe. There are no converge targets, because there is nothing to converge.
  Run `make help` for the targets; a list here would go stale.
- **`.githooks/` enforces the two rules the gate cannot** — green between commits, and the commit message shape. Opt in
  per clone with `make hooks`; each rejection prints the rule and the fix, so the hooks are the reference.
- **Collection dependencies are pinned to majors in `galaxy.yml`**, which is what a consumer resolves, and mirrored in
  `requirements.yml` for local linting. **Change both or neither.** The `ansible-core` floor lives in
  `meta/runtime.yml`, enforced at install time and re-checked by `scripts/lint.sh`.
- **Bash follows the [YSAP style guide](https://style.ysap.sh)** — the source of truth for the mechanics, don't restate
  them here. `make fmt` / `make check` apply `shfmt -i 0 -ci` and `shellcheck -x`. Two points no tool enforces:
  - **80 columns.** `shfmt` has no width flag.
  - **No `set -e`.** Errexit hides the failure that matters; check explicitly instead — `cd "$dir" || exit 1`,
    `cmd || fail=$((fail + 1))`, and a hard guard before any step unsafe to reach after a partial failure.
    `set -uo pipefail` stays: the guide rejects errexit only.
- **Search with `rg`**, never `find` or `grep`.

## Agent skills

Configuration the engineering skills read. It configures authoring _this_ repo, so it is excluded from the build.

### Issue tracker

GitHub issues on this repo's `origin`, driven by the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles, unrenamed: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See
`docs/agents/triage-labels.md`.

### Domain docs

Single-context — one `CONTEXT.md` at the root, one `docs/adr/`. See `docs/agents/domain.md`.
