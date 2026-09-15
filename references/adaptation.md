# Adapting this skill to your machine

`SKILL.md` is written to be true anywhere Codex runs. This page holds the
parts that are not: the worked examples, the diagnosis behind one piece of
advice, and the list of things you have to decide for yourself before the
skill is fully useful to you.

## What you must fill in

1. **The absolute path to `codex`.** A background runner that does not
   inherit your PATH needs it. `command -v codex` in an interactive shell.
   On the machine this skill came from it is `/home/slb/.local/bin/codex`;
   an npm global install, a Homebrew install and a distribution package all
   put it somewhere else.
2. **Two model names**, one for the ingest slot and one for the
   output-sensitive slot. Pin them somewhere your briefs can refer to by
   slot name rather than by model id, so that changing the pin is one edit.
3. **Your config, or the flags that replace it.** `codex exec` takes its
   approval policy, reasoning effort and default model from
   `$CODEX_HOME/config.toml`, not from flags. Read yours. If you are
   running Codex somewhere already isolated and want unattended execution,
   that is `approval_policy` and `sandbox_mode` in the config, or `-c`
   overrides per run — and `-s read-only` on every run that only needs to
   read.
4. **A background mechanism your harness will not interfere with.** See
   below.
5. **Whether your plan is metered, and where you read the meter.** The
   advice to avoid `--ephemeral` exists because session files under
   `$CODEX_HOME/sessions` are what usage accounting reads. If you account
   for usage some other way, keep the advice anyway for the audit trail and
   for `codex exec resume`.

## The background-launch advice, in full

`SKILL.md` says: if your harness reaps its own background children, launch
the run under something the harness does not own. That is generic. The
diagnosis that produced it is not, and it is worth reading once, because the
shape of the mistake is common.

On one Linux workstation, headless Codex runs launched through the calling
harness's own backgrounding kept dying partway. That harness culls its
background commands under a low-memory guard, and the guard reads **free**
memory rather than **available** memory. The box reported around 250 MiB
free against around 1.2 GiB genuinely available, because Linux was holding
about a gigabyte as reclaimable page cache — memory that is free the moment
anything asks for it. The machine was not short of memory at any point. A
guard was misreading it, and every multi-minute child died for it.

The fix was to stop launching the run as a child of the harness:

```bash
systemd-run --user --unit=codex-<job> --collect /path/to/runner.sh
```

A transient user unit belongs to systemd, not to the harness, so the
harness's guard never sees it as its own to cull. `--collect` makes systemd
clean the unit up after it exits, so a failed run does not leave a unit
sitting in a failed state blocking the next one with the same name.

Two consequences follow, and both cost time before they were understood:

- The unit does not inherit the interactive shell's environment, PATH
  included, so a bare `codex` in the runner exits 127 immediately. Absolute
  path, always. This is why the invocation goes in a runner file rather than
  inline on the `systemd-run` command line — a file can be read afterwards
  when the unit failed and you want to know what it actually ran.
- Whatever your harness's guard was doing, it was never a statement about
  how many workers the machine can carry. A page of this skill's own
  ancestor claimed a one-worker-at-a-time limit as if it were hardware. It
  was a workaround for the guard, written down as a fact, and it serialised
  real work for as long as nobody re-derived it. Measure the machine, and
  let the work decide the concurrency.

If you are not on Linux, or not under systemd, the generic form is what
matters: launch the long run under an owner that outlives and is invisible
to whatever might reap it. A detached process with its own session, a
`launchd` job, a terminal multiplexer session, a container — any of them
satisfies the requirement. What does not satisfy it is your harness's own
"run this in the background" affordance, if that is the thing doing the
reaping.

## The model axis, worked

The two slots in `SKILL.md` came from a fleet that pins:

- **Ingest** — `gpt-5.6-luna`. Effectively free on the ChatGPT plan, so it
  needs no justification and is not rationed. Code review, adversarial
  audits, research, transcript searching, repository-wide wording changes,
  reading millions of tokens. Work that generates leads or finds
  information, whose wrong result is discarded cheaply.
- **Output-sensitive** — `gpt-5.6-sol`. The residual default for every other
  leaf worker: implementors, bug fixers, work important enough to get right
  the first time, and every retry after an ingest-model run has failed.

Those are that fleet's nicknames and pins as of this writing, and model ids
age faster than anything else in this repository. Treat the table as the
shape of an answer, not as the answer.

The same fleet's cross-family rule, in its full form: a review never goes to
the author's own model family where another family can do it, and a draft
made by the controlling session itself is reviewed by a model from another
family, never by the small local one. Where no second family can judge the
diff and the expensive fallback is not warranted, a blind second run of the
same model is permitted — fresh brief, no sight of the author's transcript —
and is recorded as the weaker form rather than passed off as a review.

## Things deliberately left out of `SKILL.md`

- `codex exec review` and the top-level `codex review` subcommand, which run
  a review against the current repository with `--base`, `--commit` and
  `--uncommitted` selectors. They exist and they are relevant to
  cross-family review, but they are a second interface with their own flag
  set, and the skill is about driving `codex exec` as a general worker.
  Worth reading `codex exec review --help` if a review is all you want.
- `--worktree`, which runs a session in a new managed Git worktree. Useful
  for isolating a worker from your checkout; not yet exercised here.
- `--output-schema`, which constrains the final response to a JSON Schema.
  The natural next step for a caller that wants to parse a worker's verdict
  rather than read it.
- Codex's MCP, plugin, cloud and app-server surfaces. Out of scope.
