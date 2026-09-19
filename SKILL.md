---
name: codex-run
description: Run Codex CLI non-interactively from another agent or automation harness, including prompt files, workspace and sandbox settings, asynchronous jobs, failure handling, and model selection. Use for dispatching Lunas, Sols, Astras, or other gpt-family models when named.
---

# Run Codex headlessly

Use `codex exec` to run one Codex task and collect its final response. Set the
working directory explicitly and write the prompt and result to files:

```bash
codex exec \
  -C /path/to/repo \
  -m <model> \
  -s read-only \
  -o /tmp/codex-result.txt \
  < /tmp/codex-brief.md
```

Relevant options:

- `-C, --cd DIR`: working root.
- `-m, --model MODEL`: model for the run.
- `-s, --sandbox MODE`: `read-only`, `workspace-write`, or
  `danger-full-access`.
- `-o, --output-last-message FILE`: save the final response.
- `--json`: emit progress events as JSONL.
- `--add-dir DIR`: add a writable directory.
- `--skip-git-repo-check`: allow a working root outside Git.
- `--worktree`: use a managed Git worktree when isolation is useful.

Check `codex exec --help` before relying on a flag; the CLI changes over time.

## Prompts

Prefer a brief in a file. It should state:

- the task, scope, and relevant paths;
- files or directories the worker must not change;
- required checks or acceptance criteria; and
- the required final report, including anything it could not verify.

If the prompt is not an argument, `codex exec` reads it from stdin. If both a
prompt argument and piped stdin are supplied, stdin is appended as a
`<stdin>` block. Use one input method. For an argument-only prompt, redirect
stdin from `/dev/null`.

Configuration comes from `$CODEX_HOME/config.toml` unless overridden with
`-c key=value`. Review the effective model, sandbox, approval, and environment
settings before unattended execution.

Do not use `--ephemeral` for work that may need audit or resumption: it does
not persist the session. A recorded run can be continued with
`codex exec resume --last` when a matching session exists.

## Long-running jobs

`codex exec` blocks until the run finishes. Launch a multi-minute job through
a process manager or detached session if the calling harness must continue.
The process must be owned by something that will not reap it. On Linux with
systemd, use a runner script and a transient user unit:

```bash
systemd-run --user --unit=codex-<job> --collect /path/to/runner.sh
```

Call `codex` by absolute path in that script because a user unit may not have
the caller's `PATH`. See [references/adaptation.md](references/adaptation.md)
for machine-specific background-launch notes.

## Failed or refused runs

If a run stops before producing its report, inspect the workspace before
retrying:

```bash
git status --short
git diff
```

Run the relevant checks yourself, then resume the recorded session or start a
new run with a brief that accounts for the existing edits. Do not assume that
no report means no changes.

For a quota refusal, report which run was refused. If you switch models,
state the change when it affects cost, capability, or review independence.

## Choose a model

Honor a model named by the user. Otherwise, choose by the cost of an error:

- Use a cheaper long-context model for read-heavy work such as searching,
  research, audits, and reviews when its result will be checked.
- Use a stronger model for changes that will be kept, difficult fixes, or
  work where a wrong first result is expensive.

For independent review, prefer a different model family from the author when
one is available. A blind second pass from the same family can still help, but
it is weaker evidence and should be described that way.
