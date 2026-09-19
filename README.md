# codex-run

A Codex skill for running the Codex CLI as a non-interactive worker from
another agent or automation harness.

`SKILL.md` covers:

- `codex exec` invocation and the flags most useful for automation;
- prompt-file and stdin handling;
- sandbox, configuration, and session persistence;
- long-running jobs and process ownership;
- inspecting a workspace after a failed run; and
- choosing models for read-heavy work, changes, and independent review.

## Install

Place this directory in the skill directory used by your harness, keeping the
directory name `codex-run`. The harness must load `SKILL.md` as the skill
entrypoint.

The machine that launches the worker also needs an installed and authenticated
`codex` command. The skill does not install or authenticate the CLI.

## Adaptation

Before using the skill, choose model names available to your account and review
the effective `$CODEX_HOME/config.toml`. For asynchronous jobs, choose a
process manager or detached session that will outlive the calling harness.

The CLI is versioned. When a command or option is uncertain, check:

```bash
codex exec --help
```

The license is Apache-2.0; see [LICENSE](LICENSE).
