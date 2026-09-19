---
name: codex-run
description: Run Codex CLI non-interactively from another agent or automation harness, including prompt files, reviews, workspace and sandbox settings, asynchronous jobs, failure handling, structured output, and model selection.
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

The machine that launches the worker needs an installed and authenticated
`codex` command. This skill does not install or authenticate the CLI. Before
using a background service, find the executable with `command -v codex` and
put that absolute path in the runner; service managers often do not inherit
the interactive shell's `PATH`.

## Choose the execution mode

- `-C, --cd DIR` sets the worker's repository or working root.
- `-m, --model MODEL` selects the model for the run. Use a model available to
  the account, not a remembered or machine-specific name.
- `-s, --sandbox MODE` selects `read-only`, `workspace-write`, or
  `danger-full-access`. Use `read-only` when the task only needs inspection.
- `--worktree` runs the session in a new managed Git worktree. Use it when the
  worker should be isolated from the caller's checkout; do not assume its
  changes land in the primary worktree.
- `--add-dir DIR` grants an additional writable directory when the worker
  needs one outside its main root.
- `--json` streams progress events as JSONL. It is not the same as a
  structured final answer.
- `-o, --output-last-message FILE` writes the final response to a file.
- `--output-schema FILE` asks for a final response matching a JSON Schema;
  combine it with `-o` when a caller needs a machine-readable result.
- `--ephemeral` disables session persistence. Use it only when resumption and
  audit history are intentionally unnecessary.

Check the installed CLI before relying on a flag:

```bash
codex exec --help
codex exec review --help
```

Configuration, including defaults for the model, approval policy, reasoning
effort, and sandbox, comes from `$CODEX_HOME/config.toml`. Inspect the
effective configuration before unattended execution. Override a value for one
run with `-c key=value`; prefer explicit `-s`, `-m`, and `-o` flags in a
runner when those choices are part of its contract.

## Prompts and input

Prefer a brief in a file. It should state:

- the task, scope, and relevant paths;
- files or directories the worker must not change;
- required checks or acceptance criteria; and
- the required final report, including anything it could not verify.

If the prompt is not an argument, `codex exec` reads it from stdin. If both a
prompt argument and piped stdin are supplied, stdin is appended as a
`<stdin>` block. Use one input method. For an argument-only prompt, redirect
stdin from `/dev/null`.

## Review changes

Reviews are a separate, useful interface from general worker execution. Run
them from the repository being reviewed; select exactly the scope you mean:

```bash
# Uncommitted staged, unstaged, and untracked changes.
codex exec review --uncommitted - < /tmp/review-brief.md

# Changes introduced by one commit.
codex exec review --commit <sha>

# Changes from a base branch or ref.
codex exec review --base origin/main
```

The top-level `codex review` command accepts the same review selectors. Add a
custom prompt when the default review is not enough, and use `-m` with
`codex exec review` when the review model must be explicit. Review reports are
read-only with respect to the worktree: they identify findings without
modifying the reviewed tree.

For independent review, prefer a model from a different family than the
author where one is available. If no independent family is available and the
review matters, a fresh blind run of the same family is still useful, but call
it the weaker form rather than treating it as independent confirmation.

## Long-running jobs

`codex exec` blocks until the run finishes. If the calling harness reaps or
culls its own background children, launch the job under a process owner that
outlives the harness. On Linux with systemd, use a runner script and a
transient user unit:

```bash
systemd-run --user --unit=codex-<job> --collect /path/to/runner.sh
```

The runner should call Codex by its absolute path and write prompts and
results to inspectable files. `--collect` lets systemd clean up the transient
unit after it exits. On other systems, use the equivalent detached process,
`launchd` job, terminal multiplexer session, container, or other owner that
will not be reaped by the calling harness.

Do not infer a machine-wide concurrency limit from a harness's child-reaping
guard or from one low-level memory metric. Separate process ownership from
capacity: measure the machine's actual available resources and choose
concurrency from that evidence.

## Failed or refused runs

If a run stops before producing its report, inspect the workspace before
retrying:

```bash
git status --short
git diff
```

A worker may have changed files even when it did not produce a final response.
Run the relevant checks yourself, then resume the recorded session or start a
new run with a brief that accounts for the existing edits. Do not assume that
no report means no changes.

For a quota refusal, report which run was refused. If you switch models,
state the change when it affects cost, capability, or review independence.

## Choose a model

Choose by the cost of an error and by the model names currently available to
the account:

- Use a cheaper long-context model for read-heavy work such as searching,
  research, audits, reviews, and repository-wide wording changes when its
  result will be checked.
- Use a stronger output-sensitive model for changes to keep, difficult fixes,
  and retries after a failed ingest or research run.
- Keep model roles as configurable slots rather than embedding one fleet's
  nicknames or assuming that an old model ID still exists.

For independent review, prefer a different model family from the author when
one is available. Describe any same-family fallback honestly in the report.
