# sync

Running sets of rsync jobs between registered hosts. Register each machine once,
register jobs against those registered hosts, then run them all, one at a time,
by name, or by tag.

`rsync-jobs` is the tool. `defaults` holds its settings and is sourced, not run.

## Setup

```bash
cp sync/rsync-jobs.conf.example sync/rsync-jobs.conf
```

That is the whole install. `rsync-jobs.conf` is gitignored, so machine-specific
paths stay out of the repo; every setting in it is optional and commented out.
The registries are created on first write under `sync/registry/`, which is also
gitignored. `hosts.example` and `jobs.example` show the record formats if you
would rather write the registries by hand.

Settings resolve in this order, highest first: a command-line flag, an
`RV_SYNC_*` environment variable, `rsync-jobs.conf`, then `defaults`.

## Registering hosts

```bash
rsync-jobs host add nas --user weezy --address nas.lan
rsync-jobs host add media --user piders994 --address thegoldenunasinn --port 50314 --role send
rsync-jobs host add backup --address 192.168.1.40 --role recv
rsync-jobs host list
```

A host defaults to role `both` and may then be used as either end of a job.
Declaring a role makes it strict: `send` may only ever be a job source, `recv`
only a destination, and registering a job that violates that is rejected.

`local` is a built-in pseudo-host meaning a plain filesystem path. It is always
available and cannot be registered or removed. A host registered with no
`--address` also resolves to local paths.

## Registering jobs

Either as a set of options:

```bash
rsync-jobs job add --name logs-pull \
  --src media:/var/log/ --dst local:/arch/log/ \
  --opts "-az" --tags logs
```

or as one string, which is also exactly how the record is stored:

```
name|src_host:path|dst_host:path|opts|tags
```

```bash
rsync-jobs job add --string 'logs-pull|media:/var/log/|local:/arch/log/|-az|logs'
```

**Single-quote the string.** `|` is the field separator and an unquoted string
would be read by the shell as a pipeline. To avoid the quoting question
entirely, feed records on stdin instead — one per line, all validated before
anything is written, so a typo on line 9 cannot leave a half-imported registry:

```bash
rsync-jobs job add --stdin < my-jobs.txt
rsync-jobs job list --format raw > backup.txt   # round-trips through --stdin
```

Every registration is checked: the hosts must be registered, roles must allow
the direction, names must be unique, and both paths must be present. A typo'd
host name fails here rather than halfway through a transfer tonight.

## Running

```bash
rsync-jobs run all                    # everything, in parallel
rsync-jobs run logs-pull              # one job
rsync-jobs run logs-pull photos-push  # a list
rsync-jobs run --tag daily            # a tag group
rsync-jobs run --tag daily --tag logs # union of both tags

rsync-jobs run all -n                 # dry run, passes rsync -n
rsync-jobs run all -j 1               # serial
rsync-jobs run all -j 8               # eight at a time
rsync-jobs run all -v                 # stream output (implies -j 1)
rsync-jobs validate                   # check everything, run nothing
```

Jobs run in parallel by default, up to `DEFAULT_SYNC_WORKERS` at a time. Each
job gets its own log file, so concurrent output never interleaves, and the run
ends with a summary:

```
job                    result        rc   duration  log
photos-push            ok             0      4m12s  .../photos-push.log
logs-pull              FAILED        23        18s  .../logs-pull.log
db-mirror              locked         -          -  -
3 job(s): 1 ok, 1 failed, 1 locked, 0 skipped
```

Logs live in a per-run directory under `~/.local/share/rv_bashtools/logs/sync/`,
with `latest` symlinked to the most recent run. Each job log starts with the
exact rsync command that produced it, shell-quoted, so you can paste it to
reproduce a failure by hand.

Exit codes: `0` everything passed, `1` at least one job failed, `2` a usage or
validation error, `130` interrupted.

rsync's exit code 24 ("some files vanished during transfer") is reported as
`ok(24)` rather than a failure — it is routine on a live tree. Adjust
`DEFAULT_SYNC_OK_EXIT_CODES` to taste.

### Safety

`run` refuses to start if any selected job's opts delete or move data
(`--delete*`, `--del`, `--remove-source-files`) unless you pass `--yes`.
`--dry-run` never needs confirmation, so previewing is always the cheap option.

A job also takes a lock while it runs, so the same job started twice at once
reports `locked` in the second run rather than having two rsyncs fight over one
destination. That lock is per job, not per destination: if two *different* jobs
write into the same place, registration warns you, but nothing stops them.

Interrupting a run (Ctrl-C, or `kill`) stops launching new jobs, signals the
running transfers, waits `DEFAULT_SYNC_KILL_GRACE` seconds, then kills them, and
still prints a summary. Nothing is left orphaned. Add
`--partial-dir=.rsync-partial` to a job's opts if you want interrupted
transfers to resume rather than restart.

## Record formats

Both registries are pipe-delimited, one record per line, and safe to edit by
hand.

```
hosts   name|role|user|address|port|ssh_opts
jobs    name|src_host:path|dst_host:path|opts|tags
```

- Blank lines and lines starting with `#` are ignored, and your own comments
  survive `host add` / `job rm` rewrites.
- **Comments must be on their own line.** `#` is legal inside rsync opts
  (`--exclude=#*`) and inside paths, so a trailing `#...` is never stripped.
- A line with the wrong number of fields is never silently skipped: `list` and
  `validate` report it with its line number and exit non-zero, and `run`
  refuses to start at all. Pass `--force` to run anyway.
- `host:path` splits on the **first** colon, so colons inside paths are fine.
- Empty `user` or `port` means "let `~/.ssh/config` decide".

### Values cannot contain spaces

`opts` and `ssh_opts` are split on whitespace, and nothing is ever passed
through `eval`, so a single option value cannot contain a space:
`--exclude="My Documents"` will not work. This is deliberate — the alternative
is evaluating a line from a file as shell. Use `--exclude-from=/path/to/file`
for patterns with spaces, and a `Host` alias in `~/.ssh/config` for awkward ssh
options. Registration warns if it sees a quote character in `opts`.

### One side must be local

rsync cannot transfer directly between two remote machines, so a job with two
remote endpoints is rejected at registration. Stage through a local path, or run
the tool on one of the two machines.

## Testing

`sync/selftest` covers registration, validation, local transfers, concurrency,
locking, fail-fast and interrupt handling. It runs against a throwaway registry
in a temp directory and never touches the real one or the network:

```bash
sync/selftest
RV_SYNC_TEST_SSH=1 sync/selftest   # also exercise the ssh path, needs sshd-to-self
```
