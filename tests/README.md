# tests/

Not shipped — `tests` is in `galaxy.yml`'s `build_ignore`, so none of this reaches a consumer's tree.

## `golden/`

Render tests: `make test` renders every fixture through the real templates and diffs the bytes against `expected/`.
`make golden-update` accepts the current render — a separate target on purpose, because accepting an expectation should
be a deliberate act with a diff to read afterwards.

**What this is for.** `ansible-lint` checks the tasks and the tool's own validator checks the syntax; neither can see a
config that is valid and says the wrong thing. The failure mode is a rendered file that passes every validator and is
still wrong where it counts: an sshd drop-in whose effective value is not what the role claims, a journal that turns
volatile when it was supposed to survive the reboot you are debugging, an apt drop-in that does not actually disable the
date check, an rclone remote missing the flag that keeps it from listing a bucket it cannot read.

Asserts validate the consumer's _input_. Goldens validate our _output_. A correct assert can sit in front of a template
that renders the wrong bytes.

**Layout.**

```
golden/
  render.yml            # the play: find fixtures, loop, render. One run, not one per fixture
  render-one.yml        # per fixture: load defaults, load fixture, render
  steps/<role>.yml      # what to render for that role, and any pre-render derivation
  fixtures/<role>/<case>.yml
  expected/<role>/<case>/<file>
```

**Adding a case** is one fixture file plus `make golden-update`. Adding a _role_ is a `fixtures/<role>/` directory and a
`steps/<role>.yml`; nothing else changes.

Three rules the layout does not enforce:

- **Fixtures do not isolate themselves.** `render-one.yml` re-reads the role's `defaults/main.yml` before each fixture,
  and that re-read is the only thing clearing the previous fixture's values. A var the defaults do **not** declare — a
  contract var like `embed_arch_override`, or anything vaulted in the consuming repo — survives into every fixture
  sorted after it. So every fixture that cares about such a var sets it explicitly, including to empty. Without that, a
  case meant to pin one branch quietly becomes a second copy of the case before it.
- **`steps/` includes the role's own task file where a derivation is security-critical**, rather than restating it. The
  arch-to-asset derivation in `rclone` is the worked example: it is a mapping that would be trivial to copy, and a copy
  drifts from the role silently — the harness would keep rendering an asset the role stopped choosing.
- **Every value in a fixture is fake, and looks it.** The roles read real secrets from a consumer's vault; this repo has
  none and must never acquire one. Where a template renders a secret, the fixture supplies something no one could
  mistake for live (`golden-fixture-not-a-key`) and the expectation is named so no scanner reads it as a leak. The
  destination filename belongs to the task, not the template, so nothing is lost by choosing a different one here.

**One fixture per decision, not per board.** Each names the branch it exists to pin, in a comment at the top. A fixture
that sets everything at once tests nothing in particular and its diff is unreadable.
