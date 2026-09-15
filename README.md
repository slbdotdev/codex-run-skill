# codex-run-skill

A single-file agent skill for driving the Codex CLI as a headless worker
from *another* agent harness.

`SKILL.md` is the deliverable. It covers `codex exec` invocation and the
flags that matter, the stdin discipline that keeps a launch from looking
like a hang, what a brief has to contain for a worker that cannot ask you a
question, how to launch a multi-minute run so it survives, what to do when
one dies mid-flight or comes back refused for quota, and how to decide which
model a given job should go to.

## The problem it solves

Two problems, really.

**Delegation across harnesses.** Codex's non-interactive mode makes it a
usable worker for an agent that is not itself Codex. But the ergonomics are
sharp in a few specific places — a brief on stdin behaves differently from a
brief as an argument, a run with no redirect blocks on stdin and looks
hung, `--ephemeral` makes a run invisible to your own accounting, and a run
long enough to be worth delegating is long enough for a harness's background
reaper to kill it. Each of those costs an hour to rediscover. The skill
writes them down.

**Cross-family review.** A model asked to review its own output reproduces
its own blind spots, and a second instance of the same model reproduces them
again. Getting a genuinely independent read means leaving your own model
family, which means leaving your own harness. That is the main reason a
Claude Code session or a pi session reaches for `codex exec` at all, and the
skill carries the rule: a review never goes to the author's own model
family.

## Install

**Claude Code.** Clone into the personal skills directory, so that the
directory name matches the `name` in the frontmatter:

```bash
git clone https://github.com/slbdotdev/codex-run-skill \
  ~/.claude/skills/codex-run
```

Project-scoped instead of personal: `.claude/skills/codex-run/` inside the
repository. Claude Code reads the `description` in the frontmatter to decide
whether to invoke the skill, and loads the body only when it does.

**Other harnesses.** Any agent tool that reads a directory of
`SKILL.md` files takes it the same way: clone into that directory under the
name `codex-run`. For a harness with no skill mechanism at all, `SKILL.md`
is plain Markdown — paste it into the system prompt, the `AGENTS.md`, or
whatever that harness reads as standing instructions. Nothing in it is
Claude-specific.

**Prerequisite:** a working `codex` on the machine that will launch the
worker, logged in (`codex login`, or `codex login --device-auth` when
headless). The skill documents the CLI; it does not install it.

## What you must adapt

`SKILL.md` is written to be true on any machine. Five things are not, and
you have to supply them:

1. The absolute path to your `codex` binary, for any launcher that does not
   inherit your PATH.
2. Two model names — one cheap long-context model for ingest work, one
   better model for output-sensitive work. The skill gives the axis and one
   worked example; the pins are yours.
3. Your `$CODEX_HOME/config.toml`. Approval policy, sandbox mode and
   reasoning effort come from the config file, not from `codex exec` flags.
   Read yours before assuming a run is unattended.
4. A background mechanism your own harness will not reap.
5. Where you read your plan usage, if you meter it.

`references/adaptation.md` goes through all five, and includes the full
diagnosis behind the background-launch advice — a memory guard reading
*free* rather than *available* memory, killing healthy multi-minute
children on a machine that was never short of memory. That is a common
enough shape to be worth reading once even if it is not your bug.

## Verified against

`codex-cli 0.154.0`. Every flag in `SKILL.md` was checked against
`codex --help` and `codex exec --help` at that version. Codex's CLI moves;
if something has been renamed, fix the file rather than working around it.

## Licence

Apache-2.0. See `LICENSE`.
