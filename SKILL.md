---
name: codex-run
description: >-
  Drive the Codex CLI as a headless worker from another agent harness:
  `codex exec` invocation and flags, writing a brief a headless worker can act
  on, launching a run that outlives the caller, handling a run that dies or a
  quota refusal, and choosing which model to send a job to. Reading,
  inspecting, answering questions and planning about headless Codex invocation
  or worker-model selection all count, not only launching a run.
---

# Codex run

Codex ships a non-interactive mode. That makes it usable as a worker from
inside some other agent harness: you write a brief, hand it to `codex exec`,
and get a final message back in a file. Two things make that worth doing.

The first is capability routing — a cheap long-context model for work that
mostly reads, a better one for work that has to be right the first time.

The second is cross-family review. **A review never goes to the author's own
model family.** A model asked to review its own output reproduces its own
blind spots, and a second instance of the same model reproduces them again.
Reaching out of your harness into Codex is how a Claude-authored change gets
read by something that is not Claude, and the reverse.

## Invoke Codex headless

`codex exec` (alias `codex e`) runs one turn to completion and exits.

```bash
codex exec -C /path/to/repo -m <model> \
  -o /tmp/final-message.txt < /tmp/brief.md
```

The flags that matter:

- `-m`, `--model` — the model for this run.
- `-C`, `--cd` — the working root. Set it explicitly; do not inherit the
  caller's working directory by accident.
- `--skip-git-repo-check` — needed only when the working root is not inside
  a Git repository. Codex refuses to run outside one otherwise, because it
  cannot offer you a diff to undo.
- `-s`, `--sandbox` — `read-only`, `workspace-write`, or
  `danger-full-access`. Use `-s read-only` for probes, questions and
  inspection runs: a worker that cannot write cannot half-finish.
- `-o`, `--output-last-message <FILE>` — writes the agent's final message to
  a file. This is how you collect the report; without it the report exists
  only on stdout, mixed into the transcript.
- `--json` — prints events to stdout as JSONL, for a caller that follows
  progress programmatically rather than reading a transcript.
- `--add-dir <DIR>` — additional writable directories beside the working
  root.

`codex exec` has no `--ask-for-approval` flag — only the interactive command
does. Approval policy, reasoning effort and any default model come from the
config file, `$CODEX_HOME/config.toml` (default `~/.codex/config.toml`), or
per run from `-c key=value`. A managed config can supply all three, in which
case a run needs no flags for them; do not assume a config you have not read,
and prefer a per-run `-c` override to editing the config for one job.

Do not use `--ephemeral` for real work. It omits the session files under
`$CODEX_HOME/sessions`, which is where a run's record lives, so anything that
reads those files — plan-usage accounting, a later `codex exec resume`, your
own audit of what a worker actually did — will not see the run at all. The
flag is for throwaway probes and nothing else.

## Write the brief to a file

Write the brief to a file, then pipe the file:

```bash
cat > /tmp/brief.md <<'BRIEF'
...
BRIEF
codex exec -C /repo -m <model> -o /tmp/out.txt < /tmp/brief.md
```

Never pass a multi-line brief as an inline heredoc on the invocation itself,
and never assemble one inside a nested shell or `ssh` command. A file is
visible to both sides, survives the run, and can be reread when the report
disagrees with the tree.

With no prompt argument and no redirect, `codex exec` waits on stdin —
`Reading additional input from stdin...` — and a caller that expected it to
start instead looks like a hang. If you pass the prompt as an argument
rather than on stdin, still redirect: `< /dev/null`. When a prompt argument
and piped stdin are both present Codex appends stdin to the prompt as a
`<stdin>` block, which is rarely what you meant.

### What a brief needs

A headless worker cannot ask you a question. Everything it would have asked
has to be in the file already.

- **Name the method, not only the goal.** When you already know the command,
  the flag, the exact path or the file to write, put it in the brief. A
  documented trap goes in as the instruction, not as one option among
  several.
- **Say where to write and what not to touch.** Name the output path; name
  the trees that are read-only.
- **Bound the reading.** Tell it to read each file whole, once, rather than
  grepping the same file a dozen times.
- **Bound the checking.** Tell it to run the validators once, at the end,
  and name them.
- **Require honest gaps.** Tell it to state plainly what it could not
  resolve or could not verify rather than guessing and presenting the guess
  as a finding. Without that sentence you get a confident report with an
  invented fact in it.
- **Ask for the report you want.** `-o` gives you the final message, so say
  what the final message must contain, and name any file it should also
  write.

## Launch it in the background

A headless run takes minutes. Never run one in the foreground: the caller
parks, and to anyone watching that is indistinguishable from a hang. Launch
it, move on to work that does not depend on it, and answer it when it
returns. Never poll with `pgrep -f` — use whatever completion signal your
harness offers, or the run's own output file.

If your harness reaps its own background children — some do, under a memory
or process guard — a child it launched will not survive a run this long.
Launch the run under something the harness does not own instead. On Linux
with systemd, that is a transient user unit:

```bash
systemd-run --user --unit=codex-<job> --collect /path/to/runner.sh
systemctl --user is-active codex-<job>
journalctl --user -u codex-<job>
```

Put the invocation in a runner script rather than on the `systemd-run`
command line, and **call `codex` by absolute path inside it**. A systemd user
unit does not inherit your interactive shell's PATH, so a bare `codex` exits
127 immediately and the unit looks like it failed for a reason it did not.
`command -v codex` in an interactive shell gives you the path to hardcode.

A concurrency limit you inherited from a workaround is not a property of your
hardware. If you run one worker at a time because something once fell over,
measure before you keep serialising: the cause may have been a guard
misreading the machine rather than a real ceiling, and the cost of believing
it is every run you did not launch in parallel.
`references/adaptation.md` works that mistake through in full.

## When a run dies

A run that dies mid-flight is more dangerous than one that fails cleanly.
Its edits may have landed while its report never came, so the tree has
changed and you have nothing that says how.

Do not relaunch blind. Check the tree yourself — `git status`, `git diff` —
and run the validators yourself. Then decide whether to resume, redo, or
discard. `codex exec resume --last` continues a recorded session where one
exists, which is another reason not to have used `--ephemeral`.

## Quota refusals

When a run comes back refused for quota, stop sending work on that plan and
tell the human. Do not fail over silently to another model family: the human
is paying for whatever you failed over to, that model has different
strengths, and a review that quietly changed families may have just reviewed
its own author's output. Say what refused, and stop.

## Choose the worker model

A model the human names at any time wins over every default here.

The useful axis is not model size, it is what a wrong answer costs.

**Ingest work** — code review, adversarial audit, research, transcript and
log searching, repository-wide wording changes: anything that reads far more
than it writes. A wrong result is cheap, because you read the leads, discard
the bad ones, and lose only the time. Send this to the cheapest long-context
model you have that is not rationed, and send it freely.

**Output-sensitive work** — implementors, bug fixers, anything that writes
into a tree you intend to keep, anything important enough to get right the
first time, and every retry after an ingest-model run came back wrong. A
wrong result here costs a review cycle or ships a defect. Send this to your
better model and accept the price. This is the residual default: work that
is not clearly ingest belongs here.

Fill the axis in with your own models, and pin the names in one place so a
brief can say "the ingest model" and mean something. One worked example,
from the fleet this skill came out of:

| Slot | Model | Sent there |
| --- | --- | --- |
| Ingest | `gpt-5.6-luna` | review, audit, research, bulk reading |
| Output-sensitive | `gpt-5.6-sol` | implementors, fixes, retries |

Those two names are that fleet's own pins and nicknames, not Codex features.
The axis is the reusable part.

Two rules constrain the choice whatever fills it:

- A model reachable on a plan runs through its own native harness, not
  through a generic API gateway. You are already paying for the plan.
- **A review never goes to the author's own model family** where another
  family can do it. Where no other family is available, a blind second run
  of the same model — fresh brief, no sight of the author's transcript — is
  the weaker form, and gets recorded as the weaker form.

## Verified against

`codex-cli 0.154.0`. Every flag above was confirmed against `codex --help`
and `codex exec --help` at that version. Codex's CLI moves; if something
here is absent, check `codex exec --help` first and fix this file rather
than working around it.
