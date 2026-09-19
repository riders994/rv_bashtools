# lib

Shared/reusable bash functions and helpers meant to be sourced by scripts in the
other directories, rather than run directly.

- `logging` — `log_debug/info/warn/error`, honouring `LOG_LEVEL` (falling back
  to `DEFAULT_LOG_LEVEL`) and appending to `LOG_FILE` when it is set. Messages
  go to stderr so a script's stdout stays machine readable.
- `parallel` — `run_pool <workers> <fn> <item>...`, a bounded worker pool.
  Children record their own results; the pool only manages concurrency and
  teardown.
- `rsync-registry` — host and job registry parsing, validation and atomic
  writes for `sync/rsync-jobs`.
- `defaults` — shared `DEFAULT_*` settings, plus personal shell configuration.

## Careful with `defaults`

`defaults` is an interactive shell rc fragment as much as a settings file: it
defines aliases, mutates `PATH`, and starts `ssh-agent` with `ssh-add` when
`SSH_AUTH_SOCK` is unset. Sourcing it from a non-interactive script leaks a
fresh agent on every invocation, which under cron is unbounded.

Scripts that only want the `DEFAULT_*` values should default them locally
instead, the way `sync/defaults` does:

```bash
: "${DEFAULT_LOG_DIR:=${HOME}/.local/share/rv_bashtools/logs}"
```
